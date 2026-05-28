# Incident: xwing services down after IPv6-fix host changes wiped Docker's network database

| Field | Value |
|---|---|
| **Date** | 2026-05-28 |
| **Duration** | TBD pending recovery (impact began ~17:45 UTC; report opened while xwing still down) |
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
| 2026-05-23 14:24 | lucos#192 filed by SRE — xwing's `fixed-cidr-v6` uses the host's own public `/64`, same misconfiguration that caused lucos#179 on salvare |
| 2026-05-23 14:25 | Triaged Ready / Medium / owner=lucas42 (host change, can't go through CI) |
| 2026-05-28 ~17:42 | lucas42 begins host changes on xwing: edits `/etc/docker/daemon.json` to set `fixed-cidr-v6: fd42:dead:beef::/64` and `ip6tables: true`; runs `systemctl stop docker && rm -rf /var/lib/docker/network/files/ && systemctl start docker` per the fix instructions in the issue body |
| 17:44:57 | Last successful `lucos_docker_health/xwing-v4` report run — impact starts shortly after this |
| 17:45:54 | lucas42 comments "I've enacted the instructions. Waiting verification" |
| ~17:46 | First sysadmin verification probes xwing host. Daemon config correct; `/var/lib/docker/network/files/` gone; six containers `Up`. But the live-restore default kept the existing docker0 alive, so `2a01:4b00:8598:5a00::/64 dev docker0` still on the routing table |
| 17:50:23 | First sysadmin verification posted to lucos#192 — partial: "daemon.json correct, but docker0 still holding old public /64 because of live-restore" |
| ~17:51 | lucas42 runs `sudo ip link delete docker0 && sudo systemctl restart docker` to force docker0 recreation. The `ip link delete` succeeds; the restart succeeds. But because `/var/lib/docker/network/files/` was already empty from the earlier flush, the daemon comes up with **no networks at all** — default bridge included. The kernel `br-*` devices from previously-defined user networks (`lucos_router_default`, `lucos_static_media_default`, etc.) survive in the kernel but are no longer known to docker |
| ~17:51-17:52 | All running containers are now orphaned: each has a `NetworkID` in `docker inspect` for its declared `*_default` network, but `EndpointID` is empty, `IPAddress`/`Gateway` empty. `ps -ef \| grep docker-proxy` on the host returns nothing — no port-publishing processes — so external TCP connections to 80/443 on the xwing public IPs hit nothing and get "connection refused" |
| ~17:52 | Container-level `Healthy` status is largely preserved because most containers' healthchecks are container-internal (e.g. `wget http://localhost:…`) and don't traverse the broken plane. The one exception is `lucos_static_media`, whose healthcheck resolves an external hostname; it goes `unhealthy` with `wget: bad address 'l42.eu'` |
| 17:53:33 | Re-verification by sysadmin posted to lucos#192: "all green" — based on `docker ps` showing five `(healthy)` containers (the sixth, `lucos_static_media`, marked `(unhealthy)` was either missed or attributed elsewhere), and on the `docker0` public-IPv6 route being gone. `docker network ls` returning empty was explicitly labelled "pre-existing artefact of the earlier flush+live-restore" — i.e. expected harmless cleanup |
| 17:54:02 | team-lead closes lucos#192 as completed on the basis of the re-verification |
| ~18:00 | lucas42 checks `monitoring.l42.eu` and sees six failures inconsistent with the "all green" closure. Alerts team-lead |
| 18:03:32 | team-lead reopens lucos#192 with a re-open comment summarising the contradiction and dispatches SRE |
| 18:04:36 | SRE posts independent diagnosis comment on lucos#192 — `docker network ls` empty is the actual failure, not artefact; containers orphaned; recovery should be a redeploy of each xwing-hosted service to recreate networks |
| ~18:0X | `monitoringAlert` sent via Loganne by SRE to tag the investigation in the event log |
| TBD | Recovery begins — sysadmin dispatched to drive (their fingerprints on the daemon-state change) |
| TBD | All six monitoring failures clear (external `curl` 200 on every xwing public domain; `xwing-v4` and `new_files` cron paths verified via ad-hoc rerun per the standing "cron paths need end-to-end verification" rule) |
| TBD | lucos#192 re-closed against a verification standard that includes external reachability per system, not just `docker ps` Status |

---

## Analysis

### Stage 1 — The recovery procedure in lucos#192 was insufficiently specified for the live-restore case

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

---

## What Was Tried That Didn't Work

1. **The fix sequence as documented in lucos#192.** Worked correctly on a host without `live-restore: true`; on xwing (`live-restore: true`), the stop+flush+start kept the old `docker0` alive with its stale IPv6 IPAM. Sysadmin's first verification correctly caught this — the failure mode here is "incomplete instructions for the actual host configuration," not a wrong diagnosis.
2. **`ip link delete docker0 && systemctl restart docker` as the remedial step.** Forced docker0 deletion succeeded, but combined with the already-empty `/var/lib/docker/network/files/` this brought the daemon up with zero networks at all. The default `bridge` would have been lazily recreated on first container use, but the user-defined networks could not be — there is no on-host source of truth for them after the flush.
3. **Re-verification based on `docker ps` Status + IPv6 routing table check.** Reported "all green" while every container was running but unreachable. The pattern (container-internal healthchecks not exercising the failure plane) is the same one that bit on 2026-05-09, and the verification template did not learn from that.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Update lucos#192's recovery instructions to handle `live-restore: true` correctly and include a verification step that probes external reachability of every domain hosted on the affected host (not just `docker ps` / `docker network ls` / routing table). The issue itself is reopened; this is the work that closes it | lucas42/lucos#192 | Reopened — recovery in progress |
| Add to the `lucos-system-administrator` verification reference: `docker network ls` returning empty is **never** an expected steady state. The default `bridge`/`host`/`none` are always present on a working daemon. If a host change leaves them missing, the change is incomplete | TBD — file against `lucos_claude_config` or the sysadmin persona's reference file | To file |
| Add to the issue-manager / coordinator closure flow: before closing any issue whose fix touched a production host, consult `monitoring.l42.eu/api/status` and confirm no failures on systems hosted on the changed host. This would have prevented the false-positive closure at 17:54Z | TBD — file against `lucos_claude_config` (coordinator workflow) | To file |
| Document the "container-internal healthchecks don't observe the network plane" pattern in the SRE persona reference (replacing or generalising the existing `feedback_healthcheck_depth_varies.md` memory which is currently siloed in SRE memory and didn't propagate to sysadmin's verification template) | TBD | To file |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.

The outage was network-plane only; no credentials, user data, or session state was exposed or at risk.
