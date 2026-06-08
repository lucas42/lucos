# Incident: Estate-wide DNS outage (l42.eu apex zone failed to load)

| Field | Value |
|---|---|
| **Date** | 2026-06-07 |
| **Duration** | ~14 minutes (23:34 UTC to 23:48 UTC) |
| **Severity** | Complete outage (estate-wide, by name) |
| **Services affected** | All `l42.eu` services — every service name (configy, monitoring, loganne, ceol, auth, contacts, media-api, eolas, repos, …) returned SERVFAIL |
| **Detected by** | Indirect — lucos-code-reviewer flagged a "configy endpoint flap" stalling an auto-merge PR; SRE investigation found the underlying DNS outage. Notably **not** detected by monitoring (which was itself unreachable by name). |

---

## Summary

For ~14 minutes the entire `l42.eu` apex DNS zone was unresolvable: public resolvers (verified against both 8.8.8.8 and 1.1.1.1) returned SERVFAIL for every `*.l42.eu` service name, so the whole estate was effectively dark by name. The cause was a malformed generated zone file on the primary nameserver (avalon) — `dns2` was defined as both an `A` record and a `CNAME`, which BIND rejects, so it refused to load the *entire* `l42.eu` zone. The BIND container had just restarted, so there was no in-memory copy to fall back on. Service was restored by removing the conflicting record, validating with `named-checkzone`, and reloading BIND. The estate had **zero working DNS redundancy** at the time (the secondary on xwing cannot transfer zones due to a TSIG key-name mismatch), which is why a single host's zone-file error became a total outage rather than a non-event.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~23:33–23:34 | `lucos_dns_bind` on avalon restarts (inferred: a `lucos_dns` deploy). On startup it attempts to load the generated `l42.eu` zone from the shared volume. |
| 23:34:31 | BIND logs `dns2.l42.eu: CNAME and other data` → `zone l42.eu/IN: not loaded due to errors`. The apex zone is now absent; with no prior in-memory copy, every `l42.eu` name begins to SERVFAIL. (Other zones — `s.l42.eu`, `lukeblaney.co.uk`, etc. — load fine.) |
| ~23:36 | Auto-merge workflow on `lucas42/lucos_repos#410` skips: it cannot resolve/reach `configy.l42.eu` to read repo supervision status. lucos-code-reviewer notices the "configy flap". |
| 23:37:53 | `lucas42/lucos_dns#103` (the pre-existing TSIG fix PR) is created — but its CI runs land inside the outage window. |
| 23:41:48 | "Auto-merge on code reviewer approval" workflow on `lucos_dns#103` runs, exits *success* but skips the merge (configy unreachable + repo is supervised). |
| 23:45:53–23:45:58 | `convention-check` on `lucos_dns#103` runs and **fails** — it cannot reach configy/repos because DNS is down. (Non-required check; an outage casualty.) |
| ~23:44 | lucos-code-reviewer escalates the "configy flap" to SRE. Investigation begins. |
| ~23:45–23:47 | SRE confirms estate-wide SERVFAIL via public resolvers, traces the `.eu` delegation to avalon (`dns.l42.eu`, 178.32.218.44), finds BIND healthy but SERVFAILing its own zone, and locates the `CNAME and other data` load error. Broken zone file backed up. |
| 23:48:08 | SRE removes the conflicting `dns2 IN CNAME xwing.s` line, validates with `named-checkzone` (OK), SIGHUPs `named`. `zone l42.eu/IN: loaded serial 1780873200`. **Service restored.** |
| ~23:50 | Verification: `/_info` → HTTP 200 on configy/monitoring/loganne/repos/eolas; public resolvers (8.8.8.8, 1.1.1.1) resolve again. Loganne intervention event recorded. |
| 23:57:45 | `lucos_dns_sync` (config-sync) runs again now that configy is reachable, regenerates `l42.eu` (serial 1780876665) with the **already-fixed** generator. It loads cleanly — confirming the broken file was stale, and that the generator no longer produces the conflict. (This regeneration also advanced `dns2` → xwing.) |

---

## Analysis

This was not a single-fault incident. A content error in one generated file became a total estate outage because of three coinciding conditions: no validation gate before BIND served the file, no last-known-good fallback on a cold start, and no functioning secondary nameserver to absorb the primary's failure.

> All cross-repo references below use the fully-qualified `lucas42/repo#N` form because this report lives in the `lucos` repo.

### Root cause: malformed generated zone (confirmed)

The generated zone file `/etc/bind/generated-zones/l42.eu` on avalon contained two conflicting records for the same name:

```
dns2   IN A      178.32.218.44     (stale — dns2 pointing at avalon)
dns2   IN CNAME  xwing.s           (the in-flight dns2 → xwing migration)
```

A name with a `CNAME` may not have any other record type. BIND therefore rejected the **whole** `l42.eu` zone (`dns2.l42.eu: CNAME and other data` → `not loaded due to errors`). This was directly observed in BIND's logs and reproduced with `named-checkzone` against the on-disk file. The fix — deleting the `CNAME` line and reloading — restored the zone immediately, confirming the causal link.

### Contributing factor: no in-memory fallback on a cold start (confirmed)

BIND retains a zone's previous contents in memory if a *reload* fails — but the container had just (re)started, so there was no prior copy. A fresh start against a broken zone file yields a fully-absent zone, which is the difference between "a bad reload is a no-op + log line" and "the apex disappears." There is currently no startup-time validation and no last-known-good copy that BIND would prefer over a broken generated file.

### Contributing factor: provenance of the stale file (inferred, not fully confirmed)

The `dns2 IN A 178.32.218.44` + `dns2 IN CNAME xwing.s` pair is the fingerprint of a transitional state during the `dns2` → xwing secondary-DNS migration, written to the shared volume by an **older** version of the zone generator (before it grew its `dns`/`dns2` glue special-case). The current `config-sync.py` special-cases nameserver subdomains to emit `A`/`AAAA` glue and never CNAME them, and was observed regenerating a *valid* zone at 23:57:45. I did **not** directly observe what wrote the broken file or precisely what triggered the 23:34 restart (inferred to be a `lucos_dns` deploy from the container age and fresh load attempt). The load error itself, and the generator's current correctness, are both confirmed.

### Contributing factor: no working DNS redundancy (confirmed) — the real amplifier

The estate has a secondary nameserver (`lucos_dns_secondary` on xwing, intended to back `dns2.l42.eu`) precisely so that one host's BIND problem is survivable. It has **never** successfully served a zone: every AXFR/IXFR refresh from the primary fails with `tsig indicates error`, because the primary signs transfers with a key named `tsig-transfer` while the secondary expects `lucos-tsig` (BIND key names must match exactly; BADKEY fires even when the secret bytes are identical). This is documented in `lucas42/lucos_dns#103`. Consequently xwing SERVFAILs `l42.eu` and every other l42 zone, and at the time of the incident there was no second server able to answer. Had the secondary been working, avalon's broken apex would have been a degradation, not an outage.

### Contributing factor: detection gap (confirmed)

The outage was surfaced indirectly, via a stalled auto-merge PR — not by monitoring. `monitoring.l42.eu` was itself unresolvable during the window, so the system that should raise the alarm was inside the blast radius. DNS being a shared dependency of the alerting path is worth a dedicated look (see Follow-up Actions).

### Knock-on: auto-merge stalls (confirmed, self-resolving)

The estate's `code-reviewer-auto-merge` workflow reads repo supervision status from `configy.l42.eu`. During the outage that lookup failed, so approved PRs skipped merge (`lucas42/lucos_repos#410`, `lucas42/lucos_dns#103`). This was a *symptom*, not a separate fault; it cleared once DNS was restored. `lucos_repos#410` (unsupervised) can merge on a re-triggered approval; `lucos_dns#103` is on a supervised repo and correctly awaits lucas42's approval.

---

## What Was Tried That Didn't Work

- **Initial framing was wrong, then corrected.** On first diagnosis the SRE reported a "recurrence landmine — the configy source data needs fixing and the next config-sync run will overwrite the fix with a broken zone." Deeper investigation showed the opposite: the generator was *already* fixed, `systems.yaml` was correct, and the next config-sync run would (and did) regenerate a valid zone. The broken file was a stale artifact, not a live source error. Corrected before any wrong fix was attempted.
- **`rndc` was unusable for the reload.** `rndc status` / `rndc reload` from inside the BIND container returned "connection to remote host closed" (control-channel/key issue). Reload was achieved instead via `SIGHUP` to `named` (PID 8), which worked cleanly.
- **GitHub Actions re-run of the failed `convention-check` was blocked.** The lucos-site-reliability GitHub App lacks `actions: write`, so `rerun-failed-jobs` returned 403. The check is non-required and cosmetic (an outage casualty), so this was left to clear on its own rather than escalated.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Harden zone generation: `named-checkzone` before install + keep a last-known-good so a malformed/stale zone can never take down the apex (validate-before-install in config-sync; startup-time validation / last-known-good for BIND) | `lucas42/lucos_dns#104` | Open |
| Fix the TSIG key-name mismatch so the secondary can actually transfer zones and provide real redundancy (`tsig-transfer` → `lucos-tsig`) | `lucas42/lucos_dns#103` | Open (PR approved by bot; required checks green; awaits lucas42's approval — supervised repo) |
| Auto-merge stall on `lucos_repos#410` from the configy outage | `lucas42/lucos_repos#410` | Self-resolving — DNS restored; re-triggered approval should merge (in code-reviewer's loop) |
| Consider whether the alerting path should tolerate estate DNS loss (detection gap — monitoring was inside the blast radius) | Not yet filed — pending team-lead/architect input on whether to track | Open (to discuss) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

The TSIG *key names* (`tsig-transfer`, `lucos-tsig`) are identifiers, not secrets; the shared secret itself was never inspected or included. IP addresses shown (178.32.218.44 avalon, 152.37.104.10 xwing) are public-facing nameserver addresses already present in DNS.
