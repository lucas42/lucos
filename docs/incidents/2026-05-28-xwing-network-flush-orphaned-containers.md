# Incident: xwing services down after IPv6-fix host changes wiped Docker's network database

| Field | Value |
|---|---|
| **Date** | 2026-05-28 |
| **Duration** | ~1h 10m (17:45 UTC to 18:55 UTC) — comprising an initial outage to first partial recovery at 18:08–18:22 UTC, then `lucos_media_linuxplayer` remained down until a deliberate ~1m planned outage at ~18:39 UTC enabled clean daemon-restart-with-zero-containers and the second-stage redeploy of all six services |
| **Severity** | Partial outage — every container on `xwing` orphaned from its declared Docker network; all `xwing`-hosted public domains (`xwing.s.l42.eu`, `staticmedia.l42.eu`, `private.l42.eu`) refusing TCP on port 443; cascading failures on `lucos_time` and two xwing-hosted scheduled jobs (`lucos_docker_health/xwing-v4`, `lucos_media_import/new_files`) |
| **Services affected** | `lucos_router` (xwing), `lucos_static_media`, `lucos_private`, `lucos_media_import`, `lucos_media_linuxplayer`, `lucos_docker_health` (xwing-v4 only), `lucos_time` (downstream of `lucos_static_media`) |
| **Detected by** | lucas42 noticing that the `monitoring.l42.eu` view contradicted a fresh "all green" verification on lucas42/lucos#192; SRE then independently checked `/api/status` and confirmed six concurrent failures |

---

## Summary

While remediating [lucas42/lucos#192](https://github.com/lucas42/lucos/issues/192) (xwing's `/etc/docker/daemon.json` was using the host's public `/64` IPv6 prefix for the docker0 bridge — the same configuration that caused the salvare incident in [lucas42/lucos#179](https://github.com/lucas42/lucos/issues/179)), lucas42 carried out the documented fix sequence: edit `daemon.json`, flush `/var/lib/docker/network/files/`, restart docker. After an initial verification by `lucos-system-administrator` flagged that `docker0` was still holding the stale public-`/64` route because of `live-restore: true`, lucas42 ran `sudo ip link delete docker0 && sudo systemctl restart docker` to force re-creation.

The deletion succeeded — and so did far more than intended. The combined effect of the earlier flush plus the second restart left the docker daemon with **zero networks at all**, not even the default `bridge`/`host`/`none`. The kernel still held the five `br-*` bridges from the previously-flushed user-defined networks, but the daemon had no record of them. Every running container on xwing then had a phantom network reference in `docker inspect` — `NetworkID` populated, `EndpointID` empty — i.e. attached to nothing. With no docker-proxy entries publishing ports, all six xwing-hosted services became externally unreachable.

The verification that followed missed this. The `lucos-system-administrator` re-verification reported "all green" on the basis that `docker ps` showed six containers `Up X days (healthy)` and the problematic `docker0` route was gone, and explicitly labelled `docker network ls` returning empty as a "pre-existing artefact of the earlier flush+live-restore." On that basis the issue was closed at 17:54Z. lucas42 then noticed the live monitoring view contradicted the closure, SRE confirmed the outage at 18:04Z, and the issue was reopened.

The root cause of the outage is the recovery procedure documented in lucas42/lucos#192. The root cause of the *missed-detection* is that container-level `Healthy` is not proof of network reachability, especially when the failure mode is network-plane corruption that the container's own healthcheck cannot observe.

Source issue: [lucas42/lucos#192](https://github.com/lucas42/lucos/issues/192).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-23 14:24 | #192 filed by SRE — xwing's `fixed-cidr-v6` uses the host's own public `/64`, same misconfiguration that caused #179 on salvare |
| 2026-05-23 14:25 | Triaged Ready / Medium / owner=lucas42 (host change, can't go through CI) |
| 2026-05-28 ~17:42 | lucas42 begins host changes on xwing: edits `/etc/docker/daemon.json` to set `fixed-cidr-v6: fd42:dead:beef::/64` and `ip6tables: true`; runs `systemctl stop docker && rm -rf /var/lib/docker/network/files/ && systemctl start docker` per the fix instructions in the issue body |
| 17:44:57 | Last successful `lucos_docker_health/xwing-v4` report run — impact starts shortly after this |
| 17:45:54 | lucas42 comments "I've enacted the instructions. Waiting verification" |
| ~17:46 | First sysadmin verification probes xwing host. Daemon config correct; `/var/lib/docker/network/files/` gone; six containers `Up`. But the live-restore default kept the existing docker0 alive, so `2a01:4b00:8598:5a00::/64 dev docker0` still on the routing table |
| 17:50:23 | First sysadmin verification posted to #192 — partial: "daemon.json correct, but docker0 still holding old public /64 because of live-restore" |
| ~17:51 | lucas42 runs `sudo ip link delete docker0 && sudo systemctl restart docker` to force docker0 recreation. The `ip link delete` succeeds; the restart succeeds. But because `/var/lib/docker/network/files/` was already empty from the earlier flush, the daemon comes up with **no networks at all** — default bridge included. The kernel `br-*` devices from previously-defined user networks (`lucos_router_default`, `lucos_static_media_default`, etc.) survive in the kernel but are no longer known to docker |
| ~17:51-17:52 | All running containers are now orphaned: each has a `NetworkID` in `docker inspect` for its declared `*_default` network, but `EndpointID` is empty, `IPAddress`/`Gateway` empty. `ps -ef \| grep docker-proxy` on the host returns nothing — no port-publishing processes — so external TCP connections to 80/443 on the xwing public IPs hit nothing and get "connection refused" |
| ~17:52 | Container-level `Healthy` status is largely preserved because most containers' healthchecks are container-internal (e.g. `wget http://localhost:…`) and don't traverse the broken plane. The one exception is `lucos_static_media`, whose healthcheck resolves an external hostname; it goes `unhealthy` with `wget: bad address 'l42.eu'` |
| 17:53:33 | Re-verification by sysadmin posted to #192: "all green" — based on `docker ps` showing five `(healthy)` containers (the sixth, `lucos_static_media`, marked `(unhealthy)` was either missed or attributed elsewhere), and on the `docker0` public-IPv6 route being gone. `docker network ls` returning empty was explicitly labelled "pre-existing artefact of the earlier flush+live-restore" — i.e. expected harmless cleanup |
| 17:54:02 | team-lead closes #192 as completed on the basis of the re-verification |
| ~18:00 | lucas42 checks `monitoring.l42.eu` and sees six failures inconsistent with the "all green" closure. Alerts team-lead |
| 18:03:32 | team-lead reopens #192 with a re-open comment summarising the contradiction and dispatches SRE |
| 18:01:34 | `monitoringAlert` Loganne event sent by SRE to tag the investigation in the event log (preceded the diagnosis comment by ~3 minutes) |
| 18:04:36 | SRE posts independent diagnosis comment on #192 — `docker network ls` empty is the actual failure, not artefact; containers orphaned; recovery should be a redeploy of each xwing-hosted service to recreate networks |
| ~18:07 | Sysadmin triggers 6 CI redeploys via CircleCI API for the xwing-hosted services |
| 18:08:29 | `lucos_static_media v1.0.12` deployed to xwing-v4 — first network (`lucos_static_media_default`) recreated |
| 18:09:42 | `lucos_private v1.0.14` deployed |
| 18:13:35 | `lucos_router v1.0.18` deployed — three xwing public domains now externally reachable on HTTP 200 |
| 18:17:56 | `lucos_docker_health v1.0.24` deployed |
| 18:22:17 | `lucos_media_import v1.0.31` deployed; `new_files` cron resumes |
| ~18:30 | `lucos_media_linuxplayer` first-stage deploy fails: compose declares `network_mode: host` but Docker's built-in `host` network is still missing from `docker network ls`. Docker blocks manual creation of predefined networks (`docker network create host` → reserved name error) |
| ~18:34 | Sysadmin escalates daemon-restart request to lucas42 (`lucos-agent` has no NOPASSWD sudo for `systemctl`) |
| ~18:35 | First post-incident `sudo systemctl restart docker` by lucas42. Built-in `bridge`/`host`/`none` remain absent after the restart |
| ~18:36-18:38 | Sysadmin reads daemon logs (second sudo grant from lucas42) and identifies the root cause: Docker 29.4.0 emits `there are running containers, updated network configuration will not take affect` when `live-restore: true` and any containers are present. The daemon skips **all** network initialisation — including built-ins — to avoid disrupting running workloads. Every daemon restart that day had hit this because the 5 healthy containers were always present |
| ~18:39 | Sysadmin sends `plannedMaintenance` Loganne event; `docker stop`s all 6 containers (no sudo needed — three exit 0, three are SIGKILL'd at the 10s grace period). xwing is now fully down — deliberate ~1m planned outage in service of recovery |
| ~18:40 | lucas42 runs second `sudo systemctl restart docker`. With zero running containers, the daemon initialises fully: built-in `bridge`/`host`/`none` created; `docker0` recreated with `172.27.0.1/16` and `fd42:dead:beef::1/64` ULA (the prefix that #192 was about) |
| 18:40:47 | Second-stage `lucos_static_media v1.0.13` deployed |
| 18:41:40 | `lucos_private v1.0.15` deployed |
| 18:46:12 | `lucos_router v1.0.19` deployed |
| 18:48:38 | `lucos_media_linuxplayer v1.0.29` deployed — succeeds this time; `host` network is present |
| 18:50:23 | `lucos_docker_health v1.0.25` deployed |
| 18:54:18 | `lucos_media_import v1.0.32` deployed |
| 18:55:39 | Sysadmin sends `systemRecovered` Loganne event: all 6 services up, docker0 on the correct ULA prefix, no public `/64` route on docker0, external HTTP 200 on all 3 xwing public domains. SRE independently verifies and confirms |
| 18:57:09 | team-lead re-closes #192 — applied the new closure-flow STOP check (`bad0c3e`): consulted `monitoring.l42.eu/api/status` at the moment of closure, confirmed 51/51 systems healthy including all six xwing-hosted systems. Original IPv6 daemon-config fix in effect. The follow-up "recovery-instructions update" tracked in row 1 of the Follow-up Actions table remains outstanding as separate work |

---

## Analysis

### Stage 1 — The recovery procedure in #192 was insufficiently specified for the live-restore case

The original issue body's fix block was:

```bash
sudo systemctl stop docker
sudo rm -rf /var/lib/docker/network/files/
sudo systemctl start docker
```

With `live-restore: true` set on the daemon — which xwing had — `systemctl stop docker` does not stop the running containers. Their iptables rules and the existing `docker0` bridge survive the daemon stop. Flushing `/var/lib/docker/network/files/` then deletes the on-disk persistence of every network the daemon knew about. On `systemctl start docker`, the daemon comes up with an empty network database and immediately reconciles against the running containers. What that reconciliation does in detail is what bit us: the *kernel* state for `docker0` was still present (the bridge wasn't removed, only the network metadata file was), so the daemon left it alone — including its existing IPv6 IPAM. The `fixed-cidr-v6` change in `daemon.json` therefore had no effect on the actual interface, because `docker0` was never re-created. That's the state sysadmin's first verification correctly caught.

The remedial step `sudo ip link delete docker0 && sudo systemctl restart docker` then ran. This time the daemon came up with both the metadata directory empty *and* the kernel bridge gone. The default bridge was not recreated automatically — that's lazy, on first container use — so `docker network ls` returned empty including no `bridge`/`host`/`none`. And the user-defined networks (`lucos_router_default`, `lucos_static_media_default`, `lucos_private_default`, `lucos_media_import_default`, `lucos_media_linuxplayer_default`, `lucos_docker_health_default`) were not recreated either, because they were not declared anywhere the daemon could read on startup — compose files live transiently on CI runners in `/home/circleci/project` during deploy, not on the host filesystem.

The running containers, which had survived the daemon restarts thanks to `live-restore: true`, were not reattached. They retained their `NetworkID` references from before but couldn't establish endpoints against networks that no longer existed.

### Stage 2 — Container-internal healthchecks didn't detect the broken plane

Five of the six xwing containers continued to report `Up X days (healthy)` from `docker ps` throughout the outage. Their healthchecks are loopback-internal (e.g. `wget http://localhost:8080/health`) and don't traverse the docker network bridge that's now missing. `lucos_static_media` was the one exception: its healthcheck does an external DNS-requiring `wget`, which failed with `wget: bad address 'l42.eu'` — DNS resolution inside the container goes through the docker embedded DNS, which depends on the network being attached. So one out of six containers — the only one whose check happened to exercise outbound resolution — correctly flipped to `(unhealthy)`. The other five were misleadingly `(healthy)` despite being entirely cut off from the network.

This is a generalisation of the standing pattern noted in SRE memory and re-illustrated by the [2026-05-09 lucos_creds incident](2026-05-09-creds-ssh-key-crlf.md), where `lucos_creds_configy_sync`'s container healthcheck `test -p /var/log/cron.log` reported `Healthy` for hours while the SSH key it depended on was rejected outright by `libcrypto`. Docker `Healthy` is not proof of end-to-end working. It is only proof that the specific bytes the healthcheck command tests are as expected. If the healthcheck doesn't traverse the failure plane, it doesn't report on it.

### Stage 3 — Verification by `lucos-system-administrator` closed on the wrong signals

The re-verification at 17:53Z reported "all green" on the basis of three observations:

1. The IPv6 routing table no longer showed `2a01:4b00:8598:5a00::/64 dev docker0` — correct, and exactly the original issue's success criterion.
2. `docker ps` showed six containers `Up X days (healthy)` — partially correct (5/6), and not actually load-bearing on the issue's success criterion.
3. `docker network ls` returned empty — classified as "pre-existing artefact of the earlier flush+live-restore," i.e. expected harmless cleanup.

Observation 3 was the misclassification. `docker network ls` returning empty is **never** an expected steady state — even on a fresh daemon install, `bridge`, `host`, and `none` are always present. An empty list means the daemon has no networks at all, which on a host with running containers that declare networks means those containers are orphaned. The verification template should have escalated rather than dismissed.

The closure also did not include external reachability probes against the affected domains — a `curl https://xwing.s.l42.eu/_info`, `curl https://staticmedia.l42.eu/_info`, `curl https://private.l42.eu/_info`, or a glance at `monitoring.l42.eu/api/status` — any of which would have shown six concurrent failures and prevented the closure.

### Stage 4 — Detection came from a human noticing, not from the monitoring path

Six monitoring systems on `monitoring.l42.eu/api/status` were failing at the time of the closure: `xwing`, `lucos_static_media`, `lucos_private`, `lucos_time`, `lucos_docker_health` (xwing-v4 only), `lucos_media_import` (new_files only). These checks were live and red. But the issue closure path didn't consult monitoring, and the alerts hadn't surfaced anywhere that the closing party noticed in the few minutes between the verification and the closure. lucas42 spotted the contradiction within ~6 minutes of closure. Had he not, the outage would have persisted indefinitely with the ticket marked done.

### Stage 5 — Recovery extended because Docker's `live-restore` mode skips network initialisation when containers are running

The first-stage recovery (18:07–18:22 UTC) redeployed five of six services successfully. `lucos_media_linuxplayer`'s deploy then failed: its compose declares `network_mode: host`, but Docker's built-in `host` network was still missing from the daemon's database — and `docker network create host` is blocked because `host` is a reserved name.

The natural first hypothesis was "restart the daemon to repopulate the built-ins." A first `sudo systemctl restart docker` was attempted (~18:35 UTC). It did not work; the built-ins remained absent. Sysadmin then escalated for daemon-log access, and the logs revealed:

> `there are running containers, updated network configuration will not take affect`

Docker 29.4.0's `live-restore: true` deliberately skips all network initialisation — including built-in `bridge`/`host`/`none` — when any containers are running at daemon start, to avoid disrupting the live-restored workloads. The daemon was hitting this on every restart because the five recovered containers had been kept alive by the very feature that was now blocking the recovery of the sixth.

The fix was to short-circuit the live-restore protection by removing what it was protecting: `docker stop` all six containers (no sudo needed), `sudo systemctl restart docker` (sysadmin's second sudo grant from lucas42), then redeploy all six. The clean restart with zero running containers gave the daemon a free hand to initialise from scratch: built-ins created, `docker0` recreated against the new `fd42:dead:beef::/64` ULA. The redeploy sweep then re-established the user-defined networks compose-by-compose. Total deliberate-second-outage window: ~1 minute (18:39–18:40 UTC).

This stage is worth recording as its own finding because the natural "just restart the daemon" instinct fails silently in live-restore mode — and the failure isn't visible from anything outside the daemon logs, which require sudo. Without that diagnostic step, the only way out would have been progressively more aggressive guesses.

---

## What Was Tried That Didn't Work

1. **The fix sequence as documented in #192.** Worked correctly on a host without `live-restore: true`; on xwing (`live-restore: true`), the stop+flush+start kept the old `docker0` alive with its stale IPv6 IPAM. Sysadmin's first verification correctly caught this — the failure mode here is "incomplete instructions for the actual host configuration," not a wrong diagnosis.
2. **`ip link delete docker0 && systemctl restart docker` as the remedial step.** Forced docker0 deletion succeeded, but combined with the already-empty `/var/lib/docker/network/files/` this brought the daemon up with zero networks at all. The default `bridge` would have been lazily recreated on first container use, but the user-defined networks could not be — there is no on-host source of truth for them after the flush.
3. **Re-verification based on `docker ps` Status + IPv6 routing table check.** Reported "all green" while every container was running but unreachable. The pattern (container-internal healthchecks not exercising the failure plane) is the same one that bit on 2026-05-09, and the verification template did not learn from that.
4. **First post-incident daemon restart to recreate missing built-in `host` network.** Failed silently — Docker 29.4.0 with `live-restore: true` skips all network initialisation (including built-ins) when running containers are present, to avoid disrupting them. The fix required `docker stop`ping all containers first to remove the live-restore protection, then restarting the daemon clean, then redeploying. Without daemon-log access, the failure mode is invisible: `docker network ls` looks the same after a "successful" restart as it did before.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Update #192's recovery instructions to handle `live-restore: true` correctly (including the "stop-all-containers-before-restart" requirement) and include a verification step that probes external reachability of every domain hosted on the affected host (not just `docker ps` / `docker network ls` / routing table). This is the work that closes #192 | lucas42/lucos#192 | Reopened |
| Sysadmin verification template: `docker network ls` returning empty is **never** an expected steady state. Rule shipped in [lucas42/lucos_claude_config@a48fe4b](https://github.com/lucas42/lucos_claude_config/commit/a48fe4b7b9c954b0211ba2e8835d8bc38a1308a7) — Quality Control checklist point 4 now requires `curl https://<service>/_info` from outside the host for each affected HTTP service after any daemon/network change, and explicitly forbids dismissing empty `docker network ls`. Tracking sufficiency | lucas42/lucos_claude_config#95 | Open — track sufficiency |
| Coordinator closure flow: before closing any issue whose fix touched a production host, consult `monitoring.l42.eu/api/status` for failures on the affected host. Shipped as a STOP check in `references/triage-procedure.md` per [lucas42/lucos_claude_config@bad0c3e](https://github.com/lucas42/lucos_claude_config/commit/bad0c3e91aff06c8ebd5f99b35ff1eb69ab801b4) | lucas42/lucos_claude_config#96 | Closed — Done |
| Generalise the "Docker `Healthy` ≠ end-to-end working" pattern (two incidents in three weeks: 2026-05-09 lucos_creds CRLF, 2026-05-28 xwing network flush) into a cross-persona reference doc that propagates to sysadmin's verification template, code-reviewer's docker-compose review checklist, and SRE's diagnostic order. Discussion underway between SRE, developer, and architect on scope (reference doc primary; mechanical-audit complement separately scoped) | lucas42/lucos_claude_config#97 | Open — under design discussion |
| ADR-0008: formalise the lucos deployment-model property "production hosts hold no source of truth for compose-managed state, so recovery from local Docker-state corruption must round-trip through CI." The procedural counterpart is the #192 recovery-instructions update, which will cite this ADR | lucas42/lucos#199 | Draft — awaiting lucas42 sign-off |
| Loopback-healthcheck audit rule (lucos_repos convention check): warn when a service's compose `healthcheck.test` is purely a loopback probe (`localhost`/`127.0.0.1`). Warning-level, not fail, because stateless HTTP services may legitimately have loopback-only healthchecks. Mechanical complement to the judgement-based rule in #97 | lucas42/lucos_repos#404 | Open |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.

The outage was network-plane only; no credentials, user data, or session state was exposed or at risk.
