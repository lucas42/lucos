# Incident: Avalon-wide outage after Docker daemon restart left ~40 containers in orphaned containerd-task state

| Field | Value |
|---|---|
| **Date** | 2026-04-22 |
| **Duration** | ~45 minutes (13:44 UTC to ~14:30 UTC; full external recovery ~14:35 UTC) |
| **Severity** | Complete outage (all l42.eu public-facing services unreachable for ~40 min) |
| **Services affected** | All services hosted on avalon: lucos_dns, lucos_router, lucos_monitoring, lucos_loganne, lucos_authentication, all of lucos_arachne_*, lucos_contacts, lucos_docker_mirror_web/info, lucos_photos (app/worker), lucos_media_*, lucos_locations (otrecorder, otfrontend, mosquitto), lucos_scenes, lucos_time, lukeblaney_blog, lukeblaney_co_uk, semweb, tfluke, lucos_notes, lucos_creds_ui, lucos_schedule_tracker, lucos_eolas_app/web, lucos_backups, lucos_comhra, lucos_mail_smtp, and lucos_root |
| **Detected by** | User/coordinator escalation from a `lucos_docker_mirror_web` crash-loop alert; SRE investigation revealed the wider scope |
| **Source issues** | [lucas42/lucos#106](https://github.com/lucas42/lucos/issues/106), [lucas42/lucos_docker_mirror#51](https://github.com/lucas42/lucos_docker_mirror/issues/51), [lucas42/lucos_locations#69](https://github.com/lucas42/lucos_locations/issues/69) |

---

## Summary

During remediation of a Docker Hub rate-limit incident (lucas42/lucos#106), lucas42 ran `sudo systemctl restart docker` on avalon to pick up a new `registry-mirrors` config in `/etc/docker/daemon.json`. Docker's `live-restore` option was not enabled, so the restart SIGKILL'd every running container. Most containers had `restart: always` set but only ~10 auto-started again — the other ~40 were left with orphaned containerd tasks and returned `AlreadyExists` on every subsequent `docker start` attempt.

Critically, this set included `lucos_dns_bind` — the authoritative nameserver for the `l42.eu` zone — so **every l42.eu hostname went SERVFAIL at public resolvers** for the duration of the incident, amplifying a fleet-wide container-state problem into a full public-facing outage.

Recovery required a cleaner `sudo systemctl restart containerd && sudo systemctl restart docker` (in that order) to clear the orphaned task state. After the clean restart, almost all containers auto-started correctly. One container (`lucos_dns_bind`) had to be manually reconstructed via `docker run` because the SRE had `docker rm -f`'d it earlier as a diagnostic step and there was no persistent docker-compose.yml on the host to re-deploy from.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~11:43 | Estate-wide rollout kicks off ~37 concurrent CI deploys. Each deploy invokes `docker compose pull` on avalon via `DOCKER_HOST=ssh://`, which hits Docker Hub directly (no mirror configured on the production host's daemon). Avalon blows through Docker Hub's 100-pulls-per-6hr anonymous limit. |
| ~12:00–13:40 | Deploys across multiple repos begin failing at the `Pull container(s) onto remote box` step with `toomanyrequests`. Nine services stuck on old CI state; no production impact yet. |
| ~13:30 | SRE investigation concludes root cause is the missing registry-mirror on avalon's Docker daemon. Raised as [lucas42/lucos#106](https://github.com/lucas42/lucos/issues/106). |
| 13:44:50 | lucas42 applies the fix on avalon: adds `"registry-mirrors": ["https://docker.l42.eu"]` to `/etc/docker/daemon.json`, runs `sudo systemctl restart docker`. Daemon shuts down, SIGKILL's all running containers. |
| 13:44:50 | Daemon comes back up. ~10 containers successfully re-start via `restart: always`; ~40 are left in `Exited (128)` state with orphaned containerd tasks that block further `docker start` attempts. Containers that stayed down include `lucos_dns_bind`, `lucos_router`, `lucos_monitoring`, `lucos_loganne`, and `lucos_docker_mirror_info`. |
| 13:45 onwards | All `l42.eu` hostnames start SERVFAILing at public resolvers. `lucos_docker_mirror_web` enters crash-loop because its `info` upstream is unreachable. |
| ~14:00 | Coordinator escalates a `lucos_docker_mirror_web` crash-loop alert to SRE, believing it is the only affected service. |
| ~14:10 | SRE attempts diagnosis via `dig` against public resolvers and direct IP probes; observes SERVFAIL on the entire `l42.eu` zone and port 53 refused on avalon. Broader outage confirmed. |
| ~14:15 | SRE runs `docker rm -f lucos_dns_bind` as a diagnostic experiment to see whether the `AlreadyExists` state could be cleared. It could — but with no persistent docker-compose.yml on avalon, the container now has no simple recovery path. |
| ~14:17 | SRE confirms the `AlreadyExists` state is universal across ~40 stopped containers, not localised. Concludes a host-level fix is needed: either `sudo ctr -n moby task delete <id>` per container, or `sudo systemctl restart containerd && sudo systemctl restart docker`. Recommends the latter to lucas42. |
| ~14:25 | lucas42 runs `sudo systemctl restart containerd && sudo systemctl restart docker` on avalon. All orphaned tasks cleared. Containers auto-start. |
| ~14:27 | All surviving containers are `Up`. Only `lucos_dns_bind` is missing (it had been removed earlier). |
| ~14:28 | SRE runs a manual `docker run` for `lucos_dns_bind` using config reconstructed from the lucos_dns GitHub repo's `docker-compose.yml` plus `docker inspect` of the surviving `lucos_dns_sync` sibling (same network, same volume). Container comes up healthy; DNS zones notified, BIND serving queries. |
| ~14:28 | Public DNS resolution restored; `dig monitoring.l42.eu @8.8.8.8` returns the correct A record. |
| ~14:30 | SRE verifies mirror chain: `docker.l42.eu` reachable, `docker info` lists the mirror, a test `docker pull alpine:3.20` on avalon completes via the mirror. |
| ~14:30 | SRE triggers fresh CI pipelines for the 9 repos whose deploys had been blocked by the Docker Hub rate limit. |
| ~14:31 | Monitoring API settles from 49 failing checks back to 1 (an unrelated `eolas` check on `am.l42.eu`). |
| ~14:35 | Three of the 9 re-triggered pipelines have completed successfully; the rest are running or queued. Recovery effectively complete. |

---

## Analysis

This incident had four contributing factors that compounded in sequence. None of them would have been critical alone.

### Factor 1: Docker Hub rate limit exposure at deploy time

The `docker.l42.eu` pull-through cache was configured *only* in the CircleCI runner's BuildKit config, via the `lucos_deploy_orb`. The production host's Docker daemon (avalon) had no `registry-mirrors` configuration and therefore pulled every image directly from `registry-1.docker.io`, unauthenticated. Under estate-wide concurrent rollout load, that hit Docker Hub's 100-pulls-per-6hr anonymous ceiling and blocked deploys.

This was the triggering incident that made a host-level change necessary — without it, the daemon would not have been restarted today. Tracked in [lucas42/lucos#106](https://github.com/lucas42/lucos/issues/106) with the registry-mirror fix applied during this incident.

### Factor 2: `live-restore` disabled on avalon's Docker daemon

`live-restore: true` in `/etc/docker/daemon.json` instructs Docker to keep running containers alive across daemon restarts rather than terminating them. Avalon had this option unset (defaulting to `false`), so `systemctl restart docker` SIGKILL'd every container. This is the step that turned a scoped config change into a fleet-wide outage.

Had live-restore been enabled, the only disruption would have been a ~5 second loss of Docker API responsiveness during the daemon restart; running containers would have been unaffected. This was the incident's most load-bearing single factor.

### Factor 3: Orphaned containerd tasks on daemon recovery

After the SIGKILL + daemon restart, a known Docker/containerd behaviour left many containers with orphaned task records in containerd. From Docker's perspective the container was `Exited`; from containerd's, a task with the same ID still existed. Every `docker start <name>` therefore failed with `AlreadyExists: task <id>: already exists`, and `restart: always` had no effect because the daemon believed there was still a live task it shouldn't touch.

Recovery required either a surgical `sudo ctr -n moby task delete <id>` per orphan, or a full `sudo systemctl restart containerd` to clear the task namespace. This meant the SRE (in the `docker` group but not `root`) could not resolve the condition without lucas42's sudo access.

### Factor 4 (amplifier): DNS authoritative nameservers co-hosted with the affected service

`l42.eu`'s authoritative nameservers (`dns.l42.eu` + `ns1.lukeblaney.co.uk`) both live on avalon. When `lucos_dns_bind` went down, the entire zone SERVFAILed at public resolvers — not merely individual services. This made:
- Every `*.l42.eu` hostname unresolvable externally
- SSH to `avalon.s.l42.eu` impossible (name resolution failed even before the host was reached)
- Normal monitoring of the incident impossible (`monitoring.l42.eu` itself was unreachable)

Diagnosis proceeded by direct IP (`178.32.218.44`) and public DNS tracing (`dig @1.1.1.1`, `dig +trace`). Recovery continued blind to the monitoring system until DNS came back.

---

## What Was Tried That Didn't Work

- **`docker start <container>` as recovery.** The coordinator, acting on lucas42's initial understanding, asked the SRE to `docker start` the stopped containers directly. Every attempt returned `AlreadyExists: task ... already exists`. Confirmed against five containers before concluding it was universal. No combination of `docker start --detach-keys`, `docker unpause`, or `docker restart` worked — the only paths through are sudo-level `ctr` commands or a containerd restart.

- **`docker rm -f lucos_dns_bind` as a diagnostic.** The SRE ran `docker rm -f` on one container to confirm whether the `AlreadyExists` state was a containerd-task leak and could be cleared by destroying the Docker record. It did clear — but lucos uses transient-deploy directories (docker-compose.yml files only live in `/home/circleci/project` for the duration of a CI job, then vanish), so once the container record was gone, there was no `docker compose up` path to recreate it on-host. The container had to be manually reconstructed via `docker run` using config stitched from the GitHub repo + the sibling's `docker inspect`. This was a judgement error: the diagnostic was run before the universality of the issue had been established, so a single successful-but-destructive experiment turned into an unnecessary reconstruction task. See follow-up action 4 for the rule added to prevent recurrence.

- **Interpreting initial alert too narrowly.** The incident was first visible as a single-service `lucos_docker_mirror_web` crash-loop. Because that matched a previously-seen failure mode on the mirror, early framing assumed the mirror was the whole problem. External probing (`dig` at public resolvers, direct-IP `curl` to avalon) was what revealed the broader scope. In hindsight, the right first step would have been `docker ps -a` on avalon before engaging with the specific mirror symptom.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Configure `registry-mirrors` on production hosts to point at `docker.l42.eu` | [lucas42/lucos#106](https://github.com/lucas42/lucos/issues/106) | Applied to avalon during this incident; xwing/salvare pending |
| Mirror service recovered after the clean restart; kept open for the post-incident audit trail | [lucas42/lucos_docker_mirror#51](https://github.com/lucas42/lucos_docker_mirror/issues/51) | Service restored; closing after review |
| `lucos_locations_otrecorder` MQTT crash-loop | [lucas42/lucos_locations#69](https://github.com/lucas42/lucos_locations/issues/69) | Resolved incidentally by the clean restart (was a symptom of the same orphaned-task state). Keep open for the longer-term `depends_on: condition: service_healthy` improvement the issue also proposed |
| Enable `live-restore: true` on `avalon`, `xwing`, and `salvare` Docker daemons to prevent future daemon restarts from killing all running containers | [lucas42/lucos#107](https://github.com/lucas42/lucos/issues/107) | Open |
| Avoid running `docker rm -f` on stuck production containers without a confirmed recovery path; written as SRE standing rule | SRE persona memory (`feedback_no_destructive_without_recovery_path.md`) | Applied |
| Consider whether DNS authoritative nameservers should be co-hosted with services that depend on them (bootstrap/visibility during incidents) | [lucas42/lucos#109](https://github.com/lucas42/lucos/issues/109) | Open |

---

## Sensitive Findings

- [x] No — nothing in this report has been redacted.
