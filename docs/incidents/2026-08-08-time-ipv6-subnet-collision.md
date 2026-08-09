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

`lucos_time_default` was removed, and could not be recreated. This was remediation for lucas42/lucos#279, which had found that network silently ignoring its declared `enable_ipv6` for ten weeks. `lucos_time` and `lucos_dns` both declare `subnet: fd00:2::/64`, both deploy to avalon, and `lucos_dns_default` already held it — so Docker refused the allocation, all three deploy attempts failed, and `lucos_time` was left with no network and no container for just under eleven hours.

**The removal did not happen inside the deploy.** `lucos_deploy_orb`'s `deploy.yml` contains no `docker network rm`, no `docker compose down` and no `docker network prune`, and the failing deploy log shows Compose in the `Creating` state for `lucos_time_default` — which it only enters for a network that is already absent. So the destructive step was taken out of band, before the pipeline ran. That matters for prevention: a gate implemented in the orb would sit on a path this incident never took. See stage 2.

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
| *before 12:28:28* | `lucos_time_default` is removed **out of band** — not by any pipeline. Exact time and command not established; see stage 2. `lucos_time`'s container is removed with it. |
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

`lucos_time` declared `fd00:2::/64` on 2026-05-22. `lucos_dns` declared the same subnet on 2026-06-08. Both deploy to avalon. Five repos in the estate declare `fd00:*` subnets — the complete set, from each repo's `origin/main` compose file with hosts from `lucos_configy/config/systems.yaml`:

| Repo | Subnet | Host |
|---|---|---|
| `lucos_monitoring` | `fd00:1::/64` | avalon |
| `lucos_dns` | `fd00:2::/64` | avalon |
| `lucos_time` | `fd00:2::/64` | avalon — **duplicate** (now `fd00:4::/64`) |
| `lucos_backups` | `fd00:3::/64` | avalon |
| `lucos_dns_secondary` | `fd00:3::/64` | xwing |

`lucos_backups` and `lucos_dns_secondary` share `fd00:3::/64` but are on different hosts, and Docker checks pool overlap **per daemon**, so that pair does not collide today. It is on the list because the allocation key is `(host, subnet)` rather than `subnet` — which turns out to matter for what a prevention check would have to look like. Thanks to `lucos-architect` for catching that this row was missing from an earlier draft; an incident report about an allocation collision should not ship an incomplete allocation table, since this is the table readers will treat as the de facto registry.

These allocations are made per-repo, in each repo's own `docker-compose.yml`, with no shared registry and no cross-repo check. The estate has a central registry for volume names (`lucos_configy/config/volumes.yaml`) and one for hosts and domains (`lucos_configy/config/hosts.yaml`, `systems.yaml`); ULA subnets are allocated by whoever is editing a compose file that day. A duplicate is therefore not just possible but unremarkable — the two commits were three weeks apart, in different repos, and neither review had any way to see the other.

### Stage 2 — the bug being fixed was also what kept the collision latent

This is the part worth remembering. Because `lucos_time_default` was a 2024-era network that Compose never retrofitted (lucas42/lucos#279), it had no IPv6 subnet at all, so it never contended for `fd00:2::/64`. `lucos_dns_default` was recreated in June and took the subnet uncontested. The two configurations were in direct conflict for ten weeks and produced no symptom, because the defect that made the conflict harmless was the same defect whose remediation would expose it.

So the remediation was not merely risky in the ordinary way. **Removing the network was the step that converted a dormant conflict into an outage**, and the conflict was invisible to anyone reasoning from the running state — which showed one network holding `fd00:2::/64` and another with no IPv6 at all, exactly as if the allocation were fine.

#### Where the destructive step actually happened

The removal was **not** part of the deploy. Verified two ways:

- `lucas42/lucos_deploy_orb` `src/commands/deploy.yml` on `origin/main` contains no `docker network rm`, no `docker compose down`, no `docker network prune`. Its only destructive operations are `docker compose stop $HOST_SERVICES` — gated on `network_mode: host`, which `lucos_time` does not use — and `docker image prune -f`.
- The failing deploy log shows Compose in `Network lucos_time_default  Creating`, a state it only enters for a network that is already absent.

So `lucos_time_default` was gone before 12:28:28Z, removed out of band. **When, and by what command, is not established.** I do not have `sudo` on avalon and so could not read the docker daemon journal; an early attempt returned empty output, which reads exactly like "no removal event found" and was in fact "probe cannot run". The best-fitting hypothesis — offered as a hypothesis — is a manual `docker compose down`, which removes containers *and* the network in one command and accounts for both observations at once; a bare `docker network rm` would have failed with the container still attached and so needed a separate stop first. I cannot distinguish them from the evidence available.

This is load-bearing for prevention: **a gate implemented in the deploy orb would sit on a path this incident never took.**

#### What control would actually have prevented it

lucas42/lucos#279 argued the deploy-time gate should "fail loudly and require a human" rather than auto-recreate, on the grounds that recreating a network stops the containers attached to it. Keep that conclusion — but this incident strengthens it by a different argument than the one lucas42/lucos#279 made, and shows it is not sufficient on its own. Both points are `lucos-architect`'s, and they are right on both:

- **lucas42/lucos#279's argument was about cost-of-action** — a brief, planned interruption. The real hazard is that the operation is **non-atomic with no rollback**: remove, then create, and if create fails you have *neither*. That is an unbounded outage, not a brief one, and it is exactly what happened here.
- **A human gate would not have prevented this one.** A human *did* trigger the recreate, deliberately, and had no better view of subnet allocatability than the orb would have — arguably worse, since the orb runs on the host.

The control that works is a **precondition on the destructive step itself**: before removing a network, prove the declared config is allocatable on that host. It is cheap, non-destructive, and already proven — it is the same probe used during this incident to rule out IPv4 pool exhaustion:

```
docker network create --ipv6 --subnet <declared> lucos-preflight-tmp && docker network rm lucos-preflight-tmp
```

Two things worth stating alongside it, because the probe alone bounds probability rather than consequence.

**First, record the live network's config before removing it** (`docker network inspect -f '{{json .IPAM.Config}} {{.EnableIPv6}}'`), so a failed create can be rolled back to the previous working state. That is what turns an unbounded outage into a brief interruption.

**And the restore command is not the obvious one.** I proposed `docker network create --subnet <recorded> <name>`. `lucos-system-administrator` rehearsed it on a disposable project — at `lucos-architect`'s insistence that an unrehearsed rollback path is not a rollback path — and **it does not work**: `docker compose up` refuses to adopt a network it does not recognise as its own and exits 1. Compose-created networks carry identifying labels; a bare `docker network create` produces none. Verified on avalon:

```
$ docker network inspect -f '{{json .Labels}}' lucos_time_default
{"com.docker.compose.network":"default","com.docker.compose.project":"lucos_time","com.docker.compose.version":"2.27.2"}
$ docker network create sre_label_probe && docker network inspect -f '{{json .Labels}}' sre_label_probe
{}
```

The working form supplies them:

```
docker network create --subnet <recorded> \
  --label com.docker.compose.network=default \
  --label com.docker.compose.project=<project> \
  <project>_default
```

This is the single most useful thing to come out of the follow-on work, and it exists only because someone insisted on executing the recovery path rather than reasoning about it. My untested version would have failed at the exact moment it was needed — during an outage, after the destructive step, with no other way back. That is the same class of error as the incident itself: a destructive operation whose failure mode nobody had exercised.

**Second, the probe false-fails if the target network already holds its own declared subnet**, so it must never be generalised into "delete first, then probe". Per `lucos-architect`, the constructive form removes the trap structurally rather than relying on anyone remembering it: **inspect the live network first, then branch.** If it already holds the declared subnet, it is not divergent and must not be touched. Only if it is divergent — no IPv6, or a different subnet — is the declared subnet genuinely unallocated and the probe valid.

### Stage 3 — the first alert pointed at the wrong service

The incident's first alert fired at 12:33:12 and named `lucos_media_weightings: fetch-info`. `lucos_time` — the service that was actually down — did not alert until 12:40:37, seven minutes **later**. For those seven minutes the estate's only signal described a completely healthy service as unreachable.

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

### Which way Docker failed, and why that was the good outcome

It is worth being explicit that the platform behaved correctly, because the rest of this report reads as unrelieved bad news and that is not the whole picture.

Docker's IPAM overlap check is what refused the allocation. It failed **closed on isolation** and **open on availability** — we got an eleven-hour outage instead of two service stacks quietly sharing an address range. Per `lucos-security`, that ordering matters more than it looks: `lucos_dns` is a two-container stack in which `sync` has **no `ports:` mapping**, so it is unreachable from the host network, and per `references/network-topology.md` that isolation rests on Docker's per-network boundary rather than on any application-layer auth. A successful overlapping allocation would have put a question mark over exactly that boundary.

`lucos-security` hedges the mechanism explicitly, and so does this report: their working hypothesis is that a successful overlap would more likely produce further confusing packet loss (the host routing table cannot cleanly hold two identical prefixes, so NDP fails to resolve on the wrong segment) than a reliable cross-stack channel — with a narrower misdelivery case possible if both stacks allocated a container the same address. **This has not been tested against libnetwork's actual behaviour and should not be read as established.**

The reliability lesson stands regardless of which mechanism is right: an outage is the *preferable* failure here, and a prevention control should aim to stop the duplicate being written rather than to make the overlap survivable.

---

## What Was Tried That Didn't Work

**Guessing service hostnames instead of reading `systems.yaml`.** The first probes went to `time.l42.eu` and `weightings.l42.eu`. Both returned `NXDOMAIN` — which, had it gone unchecked, would have supported an entirely fictional "DNS is broken" story on a day when DNS was fine. What caught it was running a **positive control** in the same command: `root.l42.eu`, a domain I was confident existed, also came back `NXDOMAIN`. A control that fails cannot be a result. The real domains are `am.l42.eu` and `media-weighting.l42.eu`, from `lucos_configy/config/systems.yaml`.

**Grepping the broker log for `viper`.** While checking the unrelated `lucos_locations` freshness alert, a grep for `viper` across the entire `lucos_locations_mosquitto` buffer returned **zero** hits — which reads as "the phone has never connected". It has not: `viper` is the *device name in the MQTT topic*, while the client-id is `cheetah`, which appears 13 times in the same buffer. Enumerating the actual client-ids (`grep -oE 'as [A-Za-z0-9_.:-]+ ' | sort | uniq -c`) turned a false negative into a precise answer.

Both are the same mistake — accepting a clean-looking negative from a probe never shown capable of returning a positive — and both were caught by the same habit. That habit is already written into `agents/sre-ops-checks.md`; this incident is evidence it earns its place.

**Reporting a four-minute warm-up as a permanent regression.** After `lucos_monitoring_default` was recreated at 23:53:08Z and its container restarted at 23:53:14Z, `lucos_monitoring`'s self-poll went `unknown`. I measured forced-IPv6 self-fetches at 2144/5699/3431 ms against monitoring's 1s budget, correctly identified the mechanism — `{ipfamily, inet6fb4}` falls back when IPv6 *fails*, not when it is slow, so a slow-but-working path fails closed — and then **asserted, in bold, that the condition was permanent and not self-clearing.** I filed a P2 and messaged two teammates, one of whom had made the change minutes earlier.

It cleared on its own at about 23:57. The whole episode was **four minutes**; fifteen consecutive measurements afterwards ran 26–95 ms. Almost certainly ordinary settling — NDP, NAT66 conntrack, the router learning the new subnet. Retracted, and lucas42/lucos_monitoring#298 closed as invalid.

Two things make this worse than an ordinary wrong guess. **I had a background watcher polling the dashboard while I wrote the issue** — it printed `ALL GREEN` minutes later; I filed against a snapshot while a time series was being collected in the next terminal. And monitoring had logged `Warm-up: skipping alert for "lucos_monitoring" on first poll`, which I read, used to establish the restart time, and drew no conclusion from.

**For the next person recreating a network — this is the reusable part.** A container restarting onto a newly-created network will behave oddly for a few minutes. Verify immediately the two things that are immediately true (container healthy, network matches its declaration), then **leave the behavioural probe until the dust settles, and measure twice with a gap before believing anything.** Note this cuts directly against the advice I first gave: a dual-stack probe run straight after the recreate would have returned the alarming numbers and given every reason to roll back a change that was working correctly.

**A rollback recipe I asserted without testing.** I proposed `docker network create --subnet <recorded> <name>` as the restore step, in the pre-flight agreed with `lucos-system-administrator`. It does not work — `docker compose up` will not adopt a network lacking its own labels.

The first published version of this report did not carry the command, only the instruction to "restore from the recorded config" — which is arguably worse, because it points a reader at 3am towards the obvious implementation without warning them it fails. That is why this correction is a follow-up PR rather than a quiet amendment. It was caught because `lucos-architect` insisted the path be rehearsed before it counted ("an untested rollback path is not a rollback path") and `lucos-system-administrator` rehearsed it on a disposable project. The corrected command is in stage 2.

Three of my four errors tonight share a shape: I asserted something I had not executed — a duration, a persistence claim, a recovery command. The probe-discipline habit in `agents/sre-ops-checks.md` covers *observations* well and did real work today. It says nothing about **claims of the form "and if X fails, do Y"**, which are predictions about a path nobody has walked.

**Arithmetic.** The outage was reported as "~21 hours" in lucas42/lucos_time#351, in lucas42/lucos_time#352, and to team-lead, before being corrected to 10h52m. The independent figure that disproved it — `eolas-cache` reporting "38555 seconds ago", i.e. 10.7h — was quoted in the same paragraph as the wrong number and not reconciled against it. The relevant discipline ("reconcile the parts against an independent total") is already in `agents/sre-ops-checks.md` and was applied to the DNS probe forty minutes earlier in the same session, then not applied to a duration. Corrected publicly rather than silently edited, since the figure had already been relayed onward.

Everything else worked first time: the CircleCI log named the cause verbatim, and a throwaway `docker network create` on avalon (which allocated `172.16.8.0/24` cleanly) ruled out IPv4 pool exhaustion in one command.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Move `lucos_time` to `fd00:4::/64` | lucas42/lucos_time#352 | Done (merged, deployed `1.0.118`) |
| Root-cause writeup and estate subnet inventory | lucas42/lucos_time#351 | Done (closed) |
| Get the 1.0s in-band dependency probe out of `lucos_media_weightings`' `/_info` request path | lucas42/lucos_media_weightings#277 | Open |
| Register ULA subnet allocations in `lucos_configy` + enforce via a `lucos_repos` convention | lucas42/lucos#282 | Open |
| Group/de-emphasise dependent failures on the dashboard, so N shadows don't read as N incidents | lucas42/lucos_monitoring#296 | Open |
| Timeout `debug` strings must name target and configured budget — stated as a convention | lucas42/lucos_monitoring#297 | Open |
| Detect Docker networks whose live config diverges from declared compose config | lucas42/lucos#279 | Open — **see note below** |
| Restore IPv6 egress on the remaining divergent networks | lucas42/lucos#278 | Open — **see note below** |
| Mosquitto passwords-file ownership warning (unrelated, spotted during Check 4) | lucas42/lucos_locations#109 | Open (filed by `lucos-system-administrator`) |

**Note for whoever executes the remaining network recreations.** Two networks from lucas42/lucos#279's table have not yet been recreated: `lucos_monitoring_default` (avalon) and `lucos_dns_secondary_default` (xwing). Both will hit the same code path that broke `lucos_time`.

- `lucos_monitoring` declares `fd00:1::/64`. Nothing on avalon holds it, so it should succeed — but the service being recreated is *the one that would tell us if it didn't*, which makes this the worst instance to get wrong.
- `lucos_dns_secondary` declares `fd00:3::/64` on **xwing**. `lucos_backups` holds `fd00:3::/64` on **avalon**. Different hosts, so no collision — but the estate now has the same ULA subnet on two hosts by accident rather than design, which is worth a decision rather than a shrug.
- Verified by direct probe of both hosts at 2026-08-08 23:1xZ: avalon carries `fd00:2::/64` (`lucos_dns`) and `fd00:3::/64` (`lucos_backups`); xwing carries **no** `fd00:*` network at all. `fd00:4::/64` was free on both, which is why `lucos_time` got it. Re-probe before relying on this — it is a snapshot, not a registry.

- The pre-flight agreed with `lucos-system-administrator` is: sweep declared subnets across the host's full repo set → **allocatability probe** (`docker network create --ipv6 --subnet <declared> tmp && docker network rm tmp`) → **record the live network's config for rollback** → remove → redeploy → verify. `dns_secondary` on xwing first, `lucos_monitoring` on avalon last. On failure: restore from the recorded config, then stop and escalate — do **not** let CI retry blindly, since three identical failures was itself the signal on `lucos_time`.
- While `lucos_monitoring` is down, the orb's `PUT monitoring.l42.eu/suppress/$REPO` fails open (`|| true`), so **every other service's deploy silently loses alert suppression** in that window. Don't overlap deploys with it, and don't read a quiet dashboard as health — confirm from off-avalon.

### Resolved during review — the prevention design

The first draft of this report left the prevention question unfiled, on the grounds that choosing between a `lucos_repos` convention check and a `lucos_configy` allocation registry was a design call overlapping lucas42/lucos#279. **That was wrong on both counts**, and `lucos-architect` corrected it during review:

- It is **not** the same question as lucas42/lucos#279. That issue is *runtime divergence* — declared config that never took effect. This is *author-time allocation conflict* between two declarations each individually valid. Different failure, different detection point, different home. Folding them would have meant lucas42/lucos#279 could not close until a registry shipped.
- It is **not** a genuine fork, because the check-alone option does not stand up: Docker checks overlap per daemon, so the allocation key is `(host, subnet)` — a pure duplicate-detector over compose files would need the host mapping, which lives in configy. The check needs the registry regardless.

Now filed as lucas42/lucos#282, with the registry-plus-convention design and `lucos-architect`'s reasoning attributed. Recorded here rather than quietly amended, because "I deliberately didn't file this" is a claim a reader should be able to see retracted.

### Deliberately not filed

**A monitoring check for `/_info` response time against the poller's budget.** Tempting after stage 3, and rejected. It would need a per-service latency budget to be meaningful, would fire hardest on exactly the days the estate is already noisy with real alerts, and would detect a condition that is self-clearing and costs only diagnostic confusion. A line in the `/_info` spec — that the endpoint must return well inside the poller's 1s budget, and therefore must not make live calls with timeouts near it — costs one paragraph and catches the next instance at review time instead. Raised as a suggestion in lucas42/lucos_media_weightings#277 rather than as a ticket.

**Paging.** See stage 4. Not a defect, and not my call to make.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

[ ] Yes — see note below.
