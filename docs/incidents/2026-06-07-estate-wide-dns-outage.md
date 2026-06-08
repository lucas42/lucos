# Incident: Estate-wide DNS outage (l42.eu apex zone failed to load)

| Field | Value |
|---|---|
| **Date** | 2026-06-07 |
| **Duration** | ~15 minutes (23:33 UTC to 23:48 UTC) |
| **Severity** | Complete outage (estate-wide, by name) |
| **Services affected** | All `l42.eu` services — every service name (configy, monitoring, loganne, ceol, auth, contacts, media-api, eolas, repos, …) returned SERVFAIL |
| **Detected by** | Indirect — lucos-code-reviewer flagged a "configy endpoint flap" stalling an auto-merge PR; team-lead independently confirmed estate-wide resolution failure; SRE investigation found the underlying cause. Notably **not** detected by monitoring (which was itself unreachable by name). |

---

## Summary

For ~15 minutes the entire `l42.eu` apex DNS zone was unresolvable: public resolvers (verified against both 8.8.8.8 and 1.1.1.1) returned SERVFAIL for every `*.l42.eu` service name, so the whole estate was dark by name. The trigger was a deploy (`lucas42/lucos_dns#102`) that restarted BIND on the primary (avalon); on restart BIND loaded a **stale, malformed** generated zone file in which `dns2` was defined as both an `A` record and a `CNAME` — which BIND rejects, so it refused to load the *entire* `l42.eu` zone. Crucially, that deploy was itself part of the in-flight work to **build** a DNS secondary (ADR-0010), so the redundancy that would normally have absorbed a single host's failure did not yet exist — and the very change building it is what tipped it over. Service was restored by removing the conflicting record, validating with `named-checkzone`, and reloading BIND.

---

## Background: this happened *while building* the secondary, not in spite of a known gap

The lack of working DNS redundancy on 2026-06-07 was **not** a backlog item nobody had got round to. The estate was, that very evening, in the middle of standing up a second authoritative nameserver under ADR-0010 (a dedicated `lucos_dns_secondary` system running BIND `type secondary` on xwing, slaving all five managed zones from the avalon primary over TSIG-authenticated AXFR/IXFR). Three changes landed in the hour before the outage as part of that rollout:

- `lucas42/lucos_configy#212` (merged 22:53:56Z) — registered `lucos_dns_secondary` as a configy system (`domain: dns2.l42.eu`, host xwing) + its `secondaryzones` volume. This is what made `dns2 = xwing` in the data the zone generator reads.
- `lucas42/lucos_dns_secondary#6` (merged 23:03:01Z) — the secondary BIND container itself (TSIG key name `lucos-tsig` on the secondary side).
- `lucas42/lucos_dns#102` (merged 23:32:37Z) — the **primary-side** changes: `named.conf` TSIG `allow-transfer`/`also-notify` to xwing, second `dns2.l42.eu` NS records in the zones, and a `config-sync.py` change to generate `dns2` glue generically from configy. **#102's deploy is what restarted BIND and triggered the outage.**

So the redundancy was mid-construction, and two separate in-flight setup defects (a stale transitional zone file, and a TSIG key-*name* mismatch that meant the secondary couldn't yet transfer anything) combined so that the single primary had no backup at the exact moment its apex broke.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 22:53:56 | `lucas42/lucos_configy#212` merged — `dns2` becomes a configy system on xwing. From here, the (old) zone generator starts producing a *conflicting* `dns2` record set. |
| 23:03:01 | `lucas42/lucos_dns_secondary#6` merged — secondary BIND stands up on xwing (key name `lucos-tsig`). |
| 23:32:37 | `lucas42/lucos_dns#102` merged — primary-side TSIG + the **fixed** `config-sync.py` glue logic. Deploy to avalon follows. |
| ~23:33 | `lucos_dns_bind` on avalon restarts as part of the #102 deploy and loads the generated `l42.eu` from the shared volume — which still holds the **stale, pre-#102** file. |
| 23:34:31 | BIND logs `dns2.l42.eu: CNAME and other data` → `zone l42.eu/IN: not loaded due to errors`. The apex is now absent; with no prior in-memory copy and no last-known-good, every `l42.eu` name SERVFAILs. (Other zones — `s.l42.eu`, `lukeblaney.co.uk`, etc. — load fine.) |
| ~23:36 | Auto-merge on `lucas42/lucos_repos#410` skips — it can't resolve `configy.l42.eu` to read supervision status. lucos-code-reviewer notices the "configy flap". |
| ~23:38 | `lucas42/lucos_dns#103` CI fails from DNS resolution — convention-check (GitHub Actions, 23:38:00–23:38:04, couldn't resolve `repos.l42.eu`) and CircleCI build #669 (`getaddrinfo creds.l42.eu`). A second independent vantage point confirming estate-wide DNS loss. |
| 23:41:44 | `lucas42/lucos_dns#103` (the TSIG key-name fix) approved by lucos-code-reviewer. |
| ~23:44 | team-lead independently verifies all `l42.eu` names fail to resolve while github.com is fine, and escalates as a production incident. SRE investigation begins. |
| ~23:45–23:47 | SRE traces the `.eu` delegation to avalon, finds BIND healthy but SERVFAILing its own zone, and locates the `CNAME and other data` load error. Broken zone file backed up. |
| 23:48:08 | SRE removes the conflicting `dns2 IN CNAME xwing.s` line, validates with `named-checkzone` (OK), SIGHUPs `named`. `zone l42.eu/IN: loaded serial 1780873200`. **Service restored.** |
| ~23:49–23:51 | Recovery verified: public resolvers (8.8.8.8, 1.1.1.1) resolve; `/_info` → HTTP 200 across services; 3 consecutive clean probes. |
| 23:51:19 | `lucas42/lucos_repos#410` merged after re-approval (its stall was a symptom of the outage, now cleared). |
| 23:57:45 | `lucos_dns_sync` runs again now that configy is reachable and regenerates `l42.eu` (serial 1780876665) using the **already-fixed** generator — it loads cleanly, with `dns2` as A+AAAA glue to xwing and no CNAME. Confirms the broken file was stale and the generator is correct. |
| 00:03:41 (06-08) | `lucas42/lucos_dns#103` merged; avalon redeploys with the corrected TSIG key name `lucos-tsig`. |
| 00:06:15 (06-08) | Post-redeploy BIND loads serial 1780876665 and xwing's secondary now transfers all five zones cleanly with `lucos-tsig`. Verified externally: `dig l42.eu SOA @152.37.104.10` → NOERROR, serial matches the primary. **The estate finally has working DNS redundancy.** |

---

## Analysis

This was not a single-fault incident. A content error in one generated file became a total estate outage because three conditions coincided: the redundancy that should have masked it was still being built (and not yet functional), nothing validated the zone before BIND served it, and there was no last-known-good fallback on a cold start.

> All cross-repo references below use the fully-qualified `lucas42/repo#N` form because this report lives in the `lucos` repo.

### Trigger: a stale, malformed generated zone loaded on a deploy restart (confirmed)

The generated zone file `/etc/bind/generated-zones/l42.eu` on avalon contained two conflicting records for the same name:

```
dns2   IN A      178.32.218.44     (stale — dns2 pointing at avalon)
dns2   IN CNAME  xwing.s           (the dns2 → xwing migration)
```

A name with a `CNAME` may not have any other record type, so BIND rejected the **whole** `l42.eu` zone (`dns2.l42.eu: CNAME and other data` → `not loaded due to errors`). Directly observed in BIND's logs and reproduced with `named-checkzone` against the on-disk file. Deleting the `CNAME` line and reloading restored the zone immediately, confirming the causal link.

### Why the file was malformed — and why #102 is the fix, not the cause (confirmed)

The malformed file was produced by the **pre-#102** version of `config-sync.py`, not by #102. The #102 diff makes this unambiguous:

- **Before #102:** `elif subdomain == "dns":` matched only the `dns` system and *hardcoded* a second record `dns2 → <dns-host>.ipv4` (i.e. avalon). Separately, once `lucas42/lucos_configy#212` registered `lucos_dns_secondary` as a `dns2` system on xwing, the loop's generic `else` branch emitted `dns2 IN CNAME xwing.s`. Together: `dns2 IN A 178.32.218.44` **+** `dns2 IN CNAME xwing.s` — exactly the collision observed.
- **#102 changed it** to `elif subdomain in ("dns", "dns2"):`, generating each nameserver's own `A`/`AAAA` glue from its own host and dropping the hardcode. This is correct, matches ADR-0010 (`dns2` = A+AAAA glue to xwing's public IPs), and is exactly what is on disk now.

So the broken file was a **stale transitional artifact** written by the old code in the `22:53Z (#212) → 23:32Z (#102)` window. #102's deploy then restarted BIND, which loaded that stale file *before* the new (correct) `config-sync.py` could regenerate it — and the new generator then **couldn't** regenerate, because it resolves `configy.l42.eu`, which was down (see chicken-and-egg below). #102 was effectively the fix racing the broken state, and it lost the race on deploy ordering.

**Verified current state:** on-disk `l42.eu` now validates (`named-checkzone … OK`, serial 1780876665) with `dns2 IN A 152.37.104.10` + `dns2 IN AAAA …` (xwing) and no CNAME. There is no residual configy/source defect; avalon is not "one restart from re-outage."

### Contributing factor: no in-memory fallback and no last-known-good on a cold start (confirmed)

BIND retains a zone's previous contents in memory if a *reload* fails — but the container had just restarted, so there was no prior copy, and there is no on-disk last-known-good that BIND would prefer over a broken generated file. This is the difference between "a bad generation is a no-op + log line" and "the apex disappears." It is the single most important systemic gap, and the trigger that converts any future bad/stale zone into an outage.

### Contributing factor: chicken-and-egg blocked self-healing (confirmed)

`lucos_dns_sync` regenerates zones by fetching from `configy.l42.eu`. With the apex down, `configy.l42.eu` was unresolvable, so config-sync's runs failed with `NameResolutionError` and it could not replace the broken file. The system could not recover on its own; manual intervention was required to break the loop, after which config-sync immediately produced a correct zone (23:57:45).

### Contributing factor: redundancy not yet functional (confirmed) — the amplifier

The secondary being built that evening could not yet serve anything: every AXFR/IXFR from the primary failed with `tsig indicates error`, because the primary signs transfers with a key named `tsig-transfer` while the secondary expects `lucos-tsig` (BIND key names must match exactly; the *secret* matched — it was purely the name). xwing therefore held none of the zones. Tracked, with a one-line fix, in `lucas42/lucos_dns#103`. Had the secondary been functional, avalon's broken apex would have been a degradation, not an outage. This is the in-flight-setup defect referenced in the Background section, not a pre-existing neglected gap. **Resolved post-incident:** `lucas42/lucos_dns#103` merged 00:03:41Z; after the redeploy, xwing transfers cleanly with `lucos-tsig` and now answers authoritatively for all five zones (verified: `dig l42.eu SOA @152.37.104.10` → NOERROR, serial 1780876665 matching the primary). The estate now has a functioning secondary — though see the next factor for why that is not yet *externally* sufficient.

### Contributing factor: the off-site nameserver isn't off-site (confirmed)

Even with a functioning xwing secondary, the externally-visible resilience is weaker than it looks: the `.eu` parent delegation for `l42.eu` lists `dns.l42.eu` and `ns1.lukeblaney.co.uk` — but `ns1.lukeblaney.co.uk` is a CNAME to `dns.l42.eu` (avalon). So both delegation nameservers resolve to the same host, and `dns2.l42.eu` (xwing) is not in the parent delegation at all. The "off-site" name is also circularly dependent (it lives under `lukeblaney.co.uk`, which is served by the lucos primary). So although `lucas42/lucos_dns#103` made the secondary functional internally, external resolvers priming from `.eu` still only ever reach avalon. Closing this requires the registrar-side NS/glue change tracked in `lucas42/lucos#111` (which should yield a genuinely avalon-independent second NS, not just add `dns2` alongside an avalon-pointing `ns1`); the SRE follow-up `lucas42/lucos_dns#107` captures the finding and is to be reconciled with `lucas42/lucos#111`.

### Contributing factor: detection gap (confirmed)

The outage was surfaced indirectly (a stalled auto-merge PR), not by monitoring — `monitoring.l42.eu` was itself inside the blast radius. DNS being a shared dependency of the alerting path is worth a dedicated look.

---

## What Was Tried That Didn't Work

The diagnosis passed through several wrong framings before the verified cause landed — worth recording so future responders don't re-tread them:

- **"Transient DNS blip"** (initial code-reviewer read) — it was a sustained ~15-minute outage with a concrete cause, not a blip.
- **"configy endpoint flap / configy bug"** (from `lucos_repos#410`) — configy was a *victim*, not the cause. Its per-repo lookup returning data at 23:33 then nothing at 23:36 was DNS going down between the two probes, not configy flapping. Switching the auto-merge workflow to the bulk `/systems` endpoint would not have helped — both paths live on the same unresolvable domain.
- **"configy source data needs fixing / recurrence landmine"** (SRE's own first pass) — wrong: the generator was *already* fixed by #102, `systems.yaml` was correct, and the next config-sync run regenerated a valid zone unaided. Corrected before any wrong fix was attempted.
- **"#102's config-sync.py is the defect"** — wrong: the #102 diff is the fix (per-host glue, hardcode removed); the broken file predated it.

Operational dead-ends during the fix:

- **`rndc` was unusable** from the BIND container (`connection to remote host closed`). Reload was achieved via `SIGHUP` to `named` (PID 8) instead.
- **GitHub Actions re-run of the failed (non-required) `convention-check` was blocked** — the lucos-site-reliability App lacks `actions: write` (403). Left to clear on its own as it's cosmetic.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Harden zone generation: `named-checkzone` before install + keep a last-known-good so a malformed/stale zone can never take down the apex on (re)start | `lucas42/lucos_dns#104` | Open (systemic fix — the key lesson) |
| Fix the TSIG key-*name* mismatch so the secondary can transfer zones and provide real redundancy (`tsig-transfer` → `lucos-tsig`) | `lucas42/lucos_dns#103` | Done — merged 00:03:41Z; redeploy verified, xwing now slaving all five zones (l42.eu serial 1780876665 matches the primary, verified externally) |
| Give `l42.eu` genuine off-site resilience: put xwing in the `.eu` delegation with glue; stop the off-site NS being a CNAME to / circularly dependent on the primary | `lucas42/lucos_dns#107` (reconcile with `lucas42/lucos#111`, the registrar NS/glue change) | Open |
| Harden the auto-merge workflow's fail-closed behaviour to be loud rather than silent on a transient supervision-lookup failure | To be filed by lucos-architect (workflow-determinism design) | Pending — architect to file + send URL for linking |
| Auto-merge stall on `lucas42/lucos_repos#410` during the outage | `lucas42/lucos_repos#410` | Done — merged 23:51:19Z after re-approval (symptom, self-resolved with DNS) |
| Consider whether the alerting path should tolerate estate DNS loss (monitoring was inside the blast radius) | Not yet filed — pending team-lead/architect input | Open (to discuss) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

The TSIG *key names* (`tsig-transfer`, `lucos-tsig`) are identifiers, not secrets; the shared secret itself was never inspected or included. IP addresses shown (178.32.218.44 / 2001:41d0:8:dc2c::1 avalon, 152.37.104.10 / 2a01:4b00:8598:5a00:ba27:ebff:fe83:e1ee xwing) are public-facing nameserver addresses already published in DNS.
