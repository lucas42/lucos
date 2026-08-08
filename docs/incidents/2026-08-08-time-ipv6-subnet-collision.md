# Incident: lucos_time down for 10h53m — recreating its Docker network hit a duplicate IPv6 subnet

| Field | Value |
|---|---|
| **Date** | 2026-08-08 |
| **Duration** | 10h52m42s (12:28:28Z failed deploy → 23:21:10Z container healthy) |
| **Severity** | Complete outage (single service) |
| **Services affected** | `lucos_time` (down); `lucos_media_weightings` (falsely reported unreachable, functionally fine) |
| **Detected by** | Monitoring alert at 12:40:37Z; acted on by SRE ops check at ~22:50Z |
| **Originating issue** | lucas42/lucos_time#351 |

---

## Summary

A deploy intended to recreate `lucos_time_default` — remediation for lucas42/lucos#279, which had found that network silently ignoring its declared `enable_ipv6` for ten weeks — deleted the network and could not recreate it. `lucos_time` and `lucos_dns` both declare `subnet: fd00:2::/64`, both deploy to avalon, and `lucos_dns_default` already held it. Docker refused the allocation, all three deploy attempts failed, and `lucos_time` was left with no network and no container for just under eleven hours.

The fix was a one-line change moving `lucos_time` to `fd00:4::/64` (lucas42/lucos_time#352). The collision had been latent since 2026-06-08 and was only reachable because the *other* bug — the one being fixed — had been keeping the two networks apart.

---

## Timeline

All times UTC. 2026-08-08 unless stated.

| Time (UTC) | Event |
|---|---|
| *2026-05-22* | `5e98dc7` adds `enable_ipv6: true` and `subnet: fd00:2::/64` to `lucos_time`. It never takes effect — `lucos_time_default` already exists as a 2024-era IPv4-only network (lucas42/lucos#279). |
| *2026-06-08* | `dcdeef7` adds `enable_ipv6: true` and `subnet: fd00:2::/64` to `lucos_dns`. `lucos_dns_default` **is** recreated, and claims the subnet. The estate now has two repos declaring the same ULA subnet on the same host, and no way to notice. |
| *00:00 – 12:20* | Home-link packet loss incident (`2026-08-08-home-link-packet-loss.md`). Its investigation surfaces lucas42/lucos#279 and puts "recreate the three divergent networks" on the follow-up list. |
| 12:25:06 | Last successful run of the `lucos_time/eolas-cache` scheduled job. |
| **12:28:28** | Pipeline 846 starts, from commit `95201d4` "chore: trigger redeploy to recreate lucos_time_default network". |
| 12:28:5x | Deploy job 2515 fails three consecutive attempts: `failed to create network lucos_time_default: Error response from daemon: invalid pool request: Pool overlaps with other one on this address space`. The old network is gone; no container is created. **`lucos_time` is now down.** |
| 12:33:12 | First alert of the incident — and it names the **wrong service**: `1 failing check on lucos media weightings: fetch-info`. |
| 12:40:37 | `2 failing checks on lucos time: circleci, fetch-info`. Detection working as intended, ~12 minutes after the deploy window. |
| 15:25:38 | Re-alert as a third check joins: `circleci, eolas-cache, fetch-info`. `eolas-cache` trips exactly on its 3h threshold from the 12:25:06 run. |
| ~22:50 | SRE ops check begins; finds three failing systems. |
| 23:0x | Root cause confirmed from the CircleCI deploy log; lucas42/lucos_time#351 filed. |
| 23:1x | lucas42/lucos_time#352 opened (`fd00:2::/64` → `fd00:4::/64`). |
| 23:15:29 | `lucos-code-reviewer` approves. |
| 23:19:01 | lucas42 approves; PR merges. |
| **23:21:10** | `lucas42/lucos_time:1.0.118` container up and healthy. `lucos_time_default` recreated with `EnableIPv6=true subnets=fd00:4::/64 172.16.8.0/24`. |
| 23:22 | `schedule-tracker` shows `lucos_time/eolas-cache` `ok: true`, `age: 67`, `errors: 0` — the cron path has actually run. |
| ~23:25 | Monitoring reports 54/55 healthy. |

---

## Analysis

### Stage 1 — two repos claimed the same ULA subnet, and nothing could tell

`lucos_time` declared `fd00:2::/64` on 2026-05-22. `lucos_dns` declared the same subnet on 2026-06-08. Both deploy to avalon. Four repos in the estate declare `fd00:*` subnets:

| Repo | Subnet | Host |
|---|---|---|
| `lucos_monitoring` | `fd00:1::/64` | avalon |
| `lucos_dns` | `fd00:2::/64` | avalon |
| `lucos_time` | `fd00:2::/64` | avalon — **duplicate** |
| `lucos_backups` | `fd00:3::/64` | avalon |

These allocations are made per-repo, in each repo's own `docker-compose.yml`, with no shared registry and no cross-repo check. The estate has a central registry for volume names (`lucos_configy/config/volumes.yaml`) and one for hosts and domains (`lucos_configy/config/hosts.yaml`, `systems.yaml`); ULA subnets are allocated by whoever is editing a compose file that day. A duplicate is therefore not just possible but unremarkable — the two commits were three weeks apart, in different repos, and neither review had any way to see the other.

### Stage 2 — the bug being fixed was also what kept the collision latent

This is the part worth remembering. Because `lucos_time_default` was a 2024-era network that Compose never retrofitted (lucas42/lucos#279), it had no IPv6 subnet at all, so it never contended for `fd00:2::/64`. `lucos_dns_default` was recreated in June and took the subnet uncontested. The two configurations were in direct conflict for ten weeks and produced no symptom, because the defect that made the conflict harmless was the same defect whose remediation would expose it.

So the remediation was not merely risky in the ordinary way. **Deleting the network was the step that converted a dormant conflict into an outage**, and the conflict was invisible to anyone reasoning from the running state — which showed one network holding `fd00:2::/64` and another with no IPv6 at all, exactly as if the allocation were fine.

lucas42/lucos#279's own text anticipated the shape of this: it argued the deploy-time gate should "fail loudly and require a human" rather than auto-recreate, because recreating a network stops the containers attached to it. That instinct was right, and if anything this incident argues it should be *stronger* — the failure mode here was not the interruption, it was that the recreate could not succeed at all.

### Stage 3 — the first alert pointed at the wrong service

The incident's first alert, at 12:33:12, was `lucos_media_weightings: fetch-info` — seven minutes before `lucos_time` itself alerted, and describing weightings as unreachable when it was entirely healthy.

`lucos_media_weightings`' `/_info` makes an in-band call to `am.l42.eu` with a 1.0s timeout (its `time-api-reachable` check). `lucos_monitoring` allows `/_info` exactly 1.0s (`lucas42/lucos_monitoring` `src/fetcher_info.erl:238`). With `lucos_time` hard down the probe spent its full timeout, pushing `/_info` to ~1.07s, and monitoring reported `HTTP Request timed out` — i.e. "weightings is unreachable".

Measured from the `lucos_monitoring` container, before and after the fix:

| | `lucos_time` down | `lucos_time` up |
|---|---|---|
| `media-weighting.l42.eu/_info` | 1080–1261 ms | 95–155 ms |

Peers for scale, same container, same minute: `notes` 38–134 ms, `seinn` 50–63 ms, `ceol` 40–43 ms. `curl` breakdown from the avalon host put `time_connect` at 1.4 ms and `time_starttransfer` at 1120 ms — the second was spent generating the response, not on the network.

The sting is that weightings' check is *well* configured: `failThreshold: 2` to ride out blips, and `dependsOn: lucos_time` to suppress during lucos_time deploys. Neither ran, because the failure landed on monitoring's own `fetch-info` probe instead of on the declared check — and `fetch-info` cannot meaningfully carry a `dependsOn`, since it is the probe that establishes whether the service is reachable at all. A service's suppression machinery was bypassed by exactly the dependency outage it was configured for. Tracked as lucas42/lucos_media_weightings#277.

### Stage 4 — detection was fine; response took ten hours

Monitoring alerted at 12:40:37, twelve minutes after the failed deploy. Nothing about detection failed here, and it is worth stating plainly so the reader does not go looking for a monitoring gap that isn't there.

The ten hours were response latency. This estate has no paging: alerts land in Loganne and on the dashboard, and the mechanism that turns an alert into action is a scheduled SRE ops check. That is a deliberate trade for a single-operator personal estate, and mostly the right one — but it means the effective time-to-response for anything that breaks outside an ops-check window is "until the next ops check", and for a hard-down service that is worth naming rather than absorbing silently.

No follow-up is being filed for this. Adding paging to a personal estate is a decision about how the operator wants to be interrupted, not a defect, and it is not mine to make.

---

## What Was Tried That Didn't Work

**Guessing service hostnames instead of reading `systems.yaml`.** The first probes went to `time.l42.eu` and `weightings.l42.eu`. Both returned `NXDOMAIN` — which, had it gone unchecked, would have supported an entirely fictional "DNS is broken" story on a day when DNS was fine. What caught it was running a **positive control** in the same command: `root.l42.eu`, a domain I was confident existed, also came back `NXDOMAIN`. A control that fails cannot be a result. The real domains are `am.l42.eu` and `media-weighting.l42.eu`, from `lucos_configy/config/systems.yaml`.

**Grepping the broker log for `viper`.** While checking the unrelated `lucos_locations` freshness alert, a grep for `viper` across the entire `lucos_locations_mosquitto` buffer returned **zero** hits — which reads as "the phone has never connected". It has not: `viper` is the *device name in the MQTT topic*, while the client-id is `cheetah`, which appears 13 times in the same buffer. Enumerating the actual client-ids (`grep -oE 'as [A-Za-z0-9_.:-]+ ' | sort | uniq -c`) turned a false negative into a precise answer.

Both are the same mistake — accepting a clean-looking negative from a probe never shown capable of returning a positive — and both were caught by the same habit. That habit is already written into `agents/sre-ops-checks.md`; this incident is evidence it earns its place.

**Arithmetic.** The outage was reported as "~21 hours" in lucas42/lucos_time#351, in lucas42/lucos_time#352, and to team-lead, before being corrected to 10h52m. The independent figure that disproved it — `eolas-cache` reporting "38555 seconds ago", i.e. 10.7h — was quoted in the same paragraph as the wrong number and not reconciled against it. The relevant discipline ("reconcile the parts against an independent total") is already in `agents/sre-ops-checks.md` and was applied to the DNS probe forty minutes earlier in the same session, then not applied to a duration. Corrected publicly rather than silently edited, since the figure had already been relayed onward.

Everything else worked first time: the CircleCI log named the cause verbatim, and a throwaway `docker network create` on avalon (which allocated `172.16.8.0/24` cleanly) ruled out IPv4 pool exhaustion in one command.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Move `lucos_time` to `fd00:4::/64` | lucas42/lucos_time#352 | Done (merged, deployed `1.0.118`) |
| Root-cause writeup and estate subnet inventory | lucas42/lucos_time#351 | Done (closed) |
| Get the 1.0s in-band dependency probe out of `lucos_media_weightings`' `/_info` request path | lucas42/lucos_media_weightings#277 | Open |
| Detect Docker networks whose live config diverges from declared compose config | lucas42/lucos#279 | Open — **see note below** |
| Restore IPv6 egress on the remaining divergent networks | lucas42/lucos#278 | Open — **see note below** |

**Note for whoever executes the remaining network recreations.** Two networks from lucas42/lucos#279's table have not yet been recreated: `lucos_monitoring_default` (avalon) and `lucos_dns_secondary_default` (xwing). Both will hit the same code path that broke `lucos_time`.

- `lucos_monitoring` declares `fd00:1::/64`. Nothing on avalon holds it, so it should succeed — but the service being recreated is *the one that would tell us if it didn't*, which makes this the worst instance to get wrong.
- `lucos_dns_secondary` declares `fd00:3::/64` on **xwing**. `lucos_backups` holds `fd00:3::/64` on **avalon**. Different hosts, so no collision — but the estate now has the same ULA subnet on two hosts by accident rather than design, which is worth a decision rather than a shrug.
- Verified by direct probe of both hosts at 2026-08-08 23:1xZ: avalon carries `fd00:2::/64` (`lucos_dns`) and `fd00:3::/64` (`lucos_backups`); xwing carries **no** `fd00:*` network at all. `fd00:4::/64` was free on both, which is why `lucos_time` got it. Re-probe before relying on this — it is a snapshot, not a registry.

### Deliberately not filed

**A duplicate-ULA-subnet check, or a central subnet registry.** The gap is real and this incident is what it costs. But there are two quite different answers — a `lucos_repos` convention check (cheap, build-time, no runtime tax, input is four lines of YAML across four repos, and a failure is directly actionable: pick another subnet) versus a `lucos_configy` allocation registry alongside `volumes.yaml` (more work, but allocations become deliberate rather than merely non-conflicting). Picking between them is a design call that overlaps substantially with the "where does estate-wide deploy verification belong" question already open on lucas42/lucos#279, and filing a third ticket would fragment it. The full reasoning is in lucas42/lucos_time#351's Prevention section so that whoever takes lucas42/lucos#279 has it to hand.

**A monitoring check for `/_info` response time against the poller's budget.** Tempting after stage 3, and rejected. It would need a per-service latency budget to be meaningful, would fire hardest on exactly the days the estate is already noisy with real alerts, and would detect a condition that is self-clearing and costs only diagnostic confusion. A line in the `/_info` spec — that the endpoint must return well inside the poller's 1s budget, and therefore must not make live calls with timeouts near it — costs one paragraph and catches the next instance at review time instead. Raised as a suggestion in lucas42/lucos_media_weightings#277 rather than as a ticket.

**Paging.** See stage 4. Not a defect, and not my call to make.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

[ ] Yes — see note below.
