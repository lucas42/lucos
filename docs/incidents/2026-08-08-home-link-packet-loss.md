# Incident: Home broadband packet loss degrades xwing- and salvare-hosted services, amplified by Node Happy Eyeballs

| Field | Value |
|---|---|
| **Date** | 2026-08-08 |
| **Duration** | ~12h20m (00:00 UTC to ~12:20 UTC). Alerting window 00:00:56Z → 10:53:30Z; underlying packet loss persisted past the last alert and was confirmed clear at 12:41Z |
| **Severity** | Partial degradation |
| **Services affected** | `lucos_static_media`, `lucos_private` (both hosted on xwing); `salvare` unreachable from avalon for 8h35m; `lucos_time`, `lucos_media_seinn`, `lucos_backups` degraded as consumers |
| **Detected by** | SRE ops check (2026-08-08 ~10:45Z) — see "Detection was slower than it should have been" |

---

## Summary

The home broadband link that fronts **xwing** and **salvare** began dropping roughly 11–20% of packets at ~00:00Z and stayed degraded for about twelve hours. Services hosted behind that link stayed *up* but became intermittently slow — between one connection in ten and one in five to `staticmedia.l42.eu` and `private.l42.eu` needed a TCP retransmit and took an extra one to three seconds, which in practice means the occasional page or image loading a couple of seconds later than normal. `salvare` was unreachable from avalon for 8h35m. The fault is external — an ISP/home-network problem with no lucos-side fix — and it cleared on its own.

> **TBD pending lucas42's answer:** whether any of that was noticeable in use. Asked via team-lead. This is a single-user system, so the question is answerable rather than a matter of speculation, and either answer belongs here — "nothing observed" is a finding, not a gap.

The reason it is worth a report is not the loss itself but what the estate did with it: **29 monitoring alerts, of which 24 were manufactured by a Node behaviour that converts a recoverable one-second hiccup into a hard failure.** Node's Happy Eyeballs implementation abandons the IPv4 connection attempt after 500 ms, and Linux's first SYN retransmit is at ~1 s — so a single dropped SYN becomes `fetch failed` rather than a slow success. Measured during a live burst: **41% request failure against ~5% packet loss.**

That amplifier is real, is ours, and is being fixed at the root (lucas42/lucos#278). Investigating it also uncovered that three Docker networks have been silently ignoring their declared `enable_ipv6` configuration since 2026-05-22 (lucas42/lucos#279), which is *why* the amplifier had anything to bite on.

---

## Timeline

All times UTC, 2026-08-08 unless stated.

| Time (UTC) | Event |
|---|---|
| *2026-05-22* | `enable_ipv6: true` merged for `lucos_time` and `lucos_monitoring` (commit `5e98dc7`, under `lucos` ADR-0007). It never takes effect — see Analysis stage 2. |
| **00:00:56** | First alert: `lucos_private` `fetch-info`. Start of the alerting window. |
| 00:09:20 | `lucos_backups` logs the first of 72 consecutive `salvare.s.l42.eu: [Errno 110] Operation timed out`. |
| 00:09:30 | First `lucos_time` `media` alert — the check that probes `staticmedia.l42.eu` on xwing. It fires 17 times over the next 10h43m. |
| 00:14:28 | `lucos_backups` `host-tracking-failures` goes red. |
| 00:47–05:13 | Densest burst: `lucos_time`, `lucos_media_seinn`, `lucos_static_media` and `lucos_private` alert and recover repeatedly. |
| 08:43:52 | Last salvare timeout in `lucos_backups`. |
| 08:49:19 | `lucos_backups` `host-tracking-failures` recovers — **8h35m red**. |
| ~10:45 | SRE ops check begins. Monitoring reports `failing: 0` with four systems in `buffering`. |
| 10:52:35 | Final alert of the incident (`lucos_time` `media`). |
| 10:53:30 | Final recovery. Alerting window ends — but the fault has not. |
| ~11:00 | Failure reproduced from inside `lucos_time`: 22 of 100 `media` checks failing. |
| ~11:26 | Loss scoped: 9/80 TCP connects to xwing need a SYN retransmit; avalon-public, 8.8.8.8 and 1.1.1.1 are 0/80. |
| ~11:35 | Amplifier established by interleaved A/B: 41% / 4.3% / 5.7%. |
| ~11:40 | lucas42/lucos#278 filed. |
| ~12:00 | Loss still present: 8/40 connects to xwing needing a retransmit. |
| ~12:41 | **Clear.** 0/50 avalon→xwing; 0/40 home→avalon and home→Cloudflare. Monitoring 55/55 healthy, 0 failing, 0 unknown. |

---

## Analysis

### Stage 1 — the external fault: packet loss on the home WAN link

The loss is on the home broadband link itself, in **both** directions, and not on avalon's transit. Established by running the same TCP-connect probe against four destinations from inside a container on avalon:

| Destination | p50 | p90 | connects needing a SYN retransmit |
|---|---|---|---|
| `152.37.104.10:443` (xwing, via home NAT) | 20 ms | **1027 ms** | **9 / 80** |
| `178.32.218.44:443` (avalon, public) | 0 ms | 1 ms | 0 / 80 |
| `8.8.8.8:53` | 5 ms | 5 ms | 0 / 80 |
| `1.1.1.1:443` | 7 ms | 12 ms | 0 / 80 |

Slow connects clustered at **1028–1048 ms** and **3038–3065 ms** — Linux's SYN retransmit backoff (1 s, then 3 s), which is a fingerprint of dropped SYNs rather than general slowness.

Run from a host *on* the home LAN, the picture inverts as expected for a WAN-side fault: xwing-local was 0/40 while **avalon-public and Cloudflare both showed 7/40 retransmits**. Loss in both directions, on the link, not on either endpoint.

Corroborated independently by a subsystem with no connection to the HTTP checks: `lucos_backups` logged 72 consecutive SSH timeouts to `salvare` (reachable only via the same link) between 00:09:20Z and 08:43:52Z.

**`ping` is not a valid loss probe here.** ICMP to `152.37.104.10` is filtered, so `ping -c 100` reports 100% loss against a perfectly healthy path. A control against 1.1.1.1 in the same moment is what exposed that; without it the ping result would have supported a far more dramatic and entirely wrong conclusion.

### Stage 2 — the amplifier: Happy Eyeballs, and three networks that ignored their own config

Of the 29 alerts, **24 came from two Node checks** — `lucos_time`'s `media` (17) and `lucos_media_seinn`'s `media-manager` (7). Those are the amplified ones. The remaining 5 (`lucos_static_media` and `lucos_private` `fetch-info`, `lucos_private` `tls-certificate`, `lucos_backups` `host-tracking-failures`) are ordinary packet-loss failures in Erlang and Python code paths, not Node.

The Node failures presented as `TypeError: fetch failed`, `cause.code = UND_ERR_CONNECT_TIMEOUT`, at a very tight modal latency of **508–517 ms** — while a raw `net.connect()` to the same host milliseconds later succeeded in ~20 ms.

500 ms is Node's `autoSelectFamilyAttemptTimeout` default. Linux's first SYN retransmit is at ~1 s. So Node abandons the IPv4 attempt half a second before the kernel would have recovered it for free. Interleaved A/B during a live burst, n=70 per arm:

```
autoSelectFamilyAttemptTimeout=500 (default) : 29 failures  (41%)
autoSelectFamilyAttemptTimeout=3000          :  3 failures  (4.3%)
autoSelectFamily=false                       :  4 failures  (5.7%)
```

This only bites when the target is **dual-stack** *and* the caller has **no IPv6 egress** — with a single address family there is no race and the attempt timer never fires. Both conditions held: every l42.eu service subdomain is dual-stack by construction (the zone generator emits a CNAME to a host record carrying AAAA), and the calling containers had no IPv6.

**The second condition should not have been true.** `lucos_time_default`, `lucos_monitoring_default` and `lucos_dns_secondary_default` have all declared `enable_ipv6: true` since 2026-05-22 and were all running with `EnableIPv6=false`. Docker Compose attaches to a network that already exists and does not retrofit changed attributes onto it; the two avalon networks were created **2024-04-28** and never recreated. So a deliberate, reviewed, merged configuration change sat unapplied for ten weeks, with a green deploy and a clean `docker compose config` either way, and nothing detecting the divergence. That is tracked separately as lucas42/lucos#279.

### Stage 3 — detection was slower than it should have been

The fault began at 00:00:56Z and was investigated at ~10:45Z, when an ops check happened to run. Two things delayed it:

1. **`summary.failing` read `0` throughout.** The monitoring API's top-level counters showed `failing: 0` with four systems in `buffering` — i.e. failing, but not yet past their `failThreshold`. A caller reading the summary sees a zero and moves on. The per-system `status` field carried the truth.
2. **The alerts were individually unremarkable.** Each was a single check flapping and recovering within a minute. Only the *rate* was anomalous: `lucos_time`'s `media` check fired 17 times in a day against a baseline of 0–1 per **month**. Nothing compares an alert rate against its own history.

No one was paged, and nothing escalated, because each event looked like the noise we already tolerate.

---

## What Was Tried That Didn't Work

Three probes gave confidently wrong answers during this investigation. All three were caught, but only because of controls, and the pattern is the useful lesson.

1. **`ping` to the affected host** returned 100% loss. ICMP is filtered there; the path was fine. A simultaneous 1.1.1.1 control returned 0% and exposed it.

2. **Two A/B runs returned 0 failures out of 120 and 0 out of 60**, and would have "disproved" a fault that was actively firing alerts. The failure rate swung between 0% and 41% within minutes. The fix was to **interleave the arms inside a single loop** so every arm sees identical network conditions, and to check the alert duty cycle before concluding "not reproducible".

3. **`dig +short AAAA` reported five service domains as single-stack.** All were dual-stack. A CNAME'd name requires the resolver to chase into a second zone — a second round trip over the lossy link — and when that timed out, `+short` printed nothing, byte-identical to a legitimate empty answer. Two chains happened to complete, so the output was a **plausible five-and-five mix rather than uniform silence**, and a mix reads as data. Worse, a false explanation was then constructed for the pattern ("an accident of DNS authoring") and published to a ticket before being retracted. `lucos-architect` independently produced the same false negative twice the same afternoon, once reporting the near-exact inverse of the truth.

The generalisable lesson is stage-3-adjacent and has been folded into `agents/sre-ops-checks.md`: the standing rule was *"suspect a probe that returns a uniformly negative result"*, and none of these were uniformly negative. **When a probe runs over N items that can each fail independently, breakage presents as a plausible subset, not a blank.** Three cheap habits catch it: run a **known-negative** control as well as a known-positive; **reconcile the parts against an independent total**; and for DNS specifically, query the **authoritative** server rather than a recursive resolver whose timeouts are invisible to you.

---

## Incidental finding: a 108-day runaway process on avalon

Unrelated to the incident but found while examining host resource usage: a `bash` process (PID 985893, user `lucos-agent`, PPID 1) had been pinned at **99.7% CPU since 2026-04-22** — 108 days, 107 days of consumed CPU time, a full core of a four-core host. It was an orphaned agent SSH one-liner whose missing semicolons meant `kill %1` and `wait` were parsed as arguments to `sleep`, so nothing ever reaped the backgrounded job.

Killing it dropped 1-minute load average from 2.60 to 1.21.

Nothing in the estate was positioned to detect it: host CPU is not a monitoring check, and a runaway `bash` is not a container crash, so it fell between the SRE and sysadmin check sets. A prevention rule has been added to the agent instructions in `references/ssh-production.md` (never background a job inside an `ssh host "…"` one-liner; use `timeout N` in the foreground). xwing was swept and is clean.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Restore IPv6 egress on the three divergent Docker networks so a lost SYN costs ~500 ms instead of the request; then set the Node attempt timeout explicitly | lucas42/lucos#278 | Open (Ready) |
| Detect Docker networks whose live config diverges from their declared compose config, at deploy time | lucas42/lucos#279 | Open (Ready) |
| Log `error.cause.code` in `lucos_time`'s `media` check — the bare `fetch failed` string cost hours of this investigation | lucas42/lucos_time#348 | Open |
| Prevention rule for orphaned SSH background jobs | `references/ssh-production.md` (committed) | Done |
| Probe-discipline rule for the plausible-subset failure mode | `agents/sre-ops-checks.md` (committed) | Done |

### Deliberately not filed

Two things a reader might expect to see here, and why they are absent:

- **A check for the external link itself.** The fault is on infrastructure we neither own nor can fix, and it self-resolved. A monitoring check would tell us something we would learn anyway from the consumer-side alerts, at the cost of a permanent per-path config surface and a new class of alert nobody can action. The right response to an ISP fault is to notice it quickly and wait, not to instrument it.
- **An alert-rate anomaly detector** (stage 3, factor 2). It would genuinely have caught this hours earlier, and it is the most tempting follow-up in this report — but it is also a significant new capability with real false-positive risk, and it is not the cheapest fix for the actual delay. Fixing lucas42/lucos#278 removes 24 of the 29 alerts at source. If a comparable alert storm recurs *after* that lands, this becomes worth revisiting with evidence rather than in anticipation.

What *has* been recorded instead is the concrete trap: **`summary.failing == 0` is not "all well"** — read the per-system `status` field, because `buffering` means failing-but-not-yet-past-threshold. That is now in the SRE ops-check notes.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

[ ] Yes — see note below.
