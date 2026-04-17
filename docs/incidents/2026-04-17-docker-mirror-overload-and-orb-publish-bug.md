# Incident: Docker mirror collapses under concurrent CI load (with an earlier orb publish bug)

| Field | Value |
|---|---|
| **Date** | 2026-04-17 |
| **Duration** | Two overlapping failures on the same day's orb rollout: orb publish bug ~16:22–18:53 UTC (~2h 30m, 1 service affected); mirror overload ~19:06–~19:22 UTC acute, ~17 min follow-up to re-run 6 builds (6 services affected) |
| **Severity** | Partial degradation — blocked deploys, no services went down (existing containers continued serving traffic) |
| **Services affected** | lucos_contacts (orb bug); lucos_media_seinn, lucos_media_metadata_manager, lucos_media_metadata_api, lucos_media_weightings, lucos_repos, lucos_media_linuxplayer (mirror overload) |
| **Detected by** | Monitoring alerts on `circleci` checks across affected services |

Source issues: lucas42/lucos_deploy_orb#119 (publish-docker `docker tag` bug, closed by #120), lucas42/lucos_docker_mirror#19 (mirror capacity, open), lucas42/lucos_deploy_orb#122 (mirror fallback, open).

---

## Summary

Today's orb rollout introduced a brand-new pull-through Docker Hub cache at `docker.l42.eu` (`lucos_docker_mirror`) and routed all CI builds through it. Two separate failures followed. First, a `docker tag` bug in `publish-docker.yml` broke the `lucos/build-amd64` job for every main-branch build for ~2.5 hours until lucas42/lucos_deploy_orb#120 shipped a fix. Second, once monitoring alerts had flagged 8 stale CircleCI checks and the coordinator re-triggered the 8 pipelines in parallel, the newly-deployed mirror's gunicorn proxy OOM-ed and dropped blob transfers mid-stream, failing 6 of the 8 builds with a mixture of connect-timeouts and BuildKit digest-mismatch errors. The mirror was restored by restarting its web container; the 6 failed pipelines were then re-run sequentially and all succeeded. No user-facing service went down at any point.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 16:13 | Commit `39dcac9` lands on `lucos_deploy_orb` main — introduces BuildKit `registry-mirrors` config pointing at `docker.l42.eu` (lucas42/lucos_deploy_orb#118). The companion change to `publish-docker-multiplatform.yml` is correct, but the change to `publish-docker.yml` leaves behind a `docker tag` call against an image that was never loaded onto the host daemon |
| 16:21 | lucas42/lucos_deploy_orb#118 merged; broken orb version publishes |
| 16:22 | lucos_contacts main-branch pipeline `#a7438b50` runs against the new orb; `lucos/build-amd64` fails on the `Docker Tag & Push (Latest)` step with "No such image". No other repo pushed to main during this window, so only lucos_contacts is affected |
| 18:51 | Fix committed (`dbc8603`) — replace `docker tag` + `docker push` with `docker buildx imagetools create` (server-side manifest tag) |
| 18:53 | lucas42/lucos_deploy_orb#120 merged; fixed orb version publishes |
| 18:55 | lucos_contacts re-triggered; pipeline `#cce080e6` on the same commit succeeds, confirming the orb fix |
| 19:06 | Coordinator re-triggers 8 main-branch pipelines in parallel across repos whose `circleci` monitoring check had been red (seinn, locations, metadata_manager, repos, creds, linuxplayer, metadata_api, weightings) |
| 19:06–19:11 | `lucos_docker_mirror_web` gunicorn workers start getting SIGKILL-ed ("Perhaps out of memory?"); workers also start tripping the 30s timeout mid-blob. Registry logs show HTTP 500s with `broken pipe` after transferring 46–91 MB of a blob. External requests to `https://docker.l42.eu/_info` begin timing out at 10s |
| 19:11–19:16 | 6 of 8 re-triggered pipelines fail — 2 with `Docker Login (mirror)` `context deadline exceeded`; 1 (`lucos_media_seinn`) with BuildKit `failed to compute cache key: ... unexpected commit digest` after receiving a truncated node blob from the mirror; 3 others with similar patterns |
| 19:16 | SRE investigation opens. Baseline monitoring snapshot captured. Root cause identified: the mirror is saturated, not a new orb bug. Internal check from avalon (`curl http://localhost:8038/_info`) returns instantly, confirming the problem is worker starvation not network/TLS |
| 19:21 | Decision: leave the decommissioned GHCR mirror alone and fix `docker.l42.eu` capacity directly |
| 19:22 | `docker restart lucos_docker_mirror_web` on avalon. External `https://docker.l42.eu/_info` immediately returns HTTP 200 in 0.37s |
| 19:22 | Follow-up issues raised: lucas42/lucos_docker_mirror#19 (worker sizing / gunicorn timeout) and lucas42/lucos_deploy_orb#122 (graceful fallback when mirror is unavailable) |
| 19:22 | The 2 pipelines (lucos_creds, lucos_locations) that were still running when the restart completed both go green |
| 19:21–19:38 | The 6 failed pipelines are re-triggered **sequentially, one at a time** (script waits for each workflow to reach a terminal status before starting the next). All 6 succeed |
| 19:38 | All affected services green in monitoring. Incident resolved |

---

## Analysis

### Stage 1: `publish-docker.yml` `docker tag` bug (the morning fault)

#### Contributing factor: inconsistent fix across two near-identical orb commands

The orb has two parallel publish commands — `publish-docker` (used by `build-amd64`) and `publish-docker-multiplatform` (used by `build-multiplatform`). A previous commit had already migrated `publish-docker-multiplatform.yml` to use `docker buildx imagetools create` for the `:latest` tag push, correctly recognising that the buildx docker-container driver never loads the built image into the host daemon. The change to `publish-docker.yml` in lucas42/lucos_deploy_orb#118 left behind the older `docker tag` + `docker push` pattern, which fails immediately with "No such image" whenever the builder is the docker-container driver (i.e. always, for these jobs).

The bug was shipped to production because no pre-publish test exists for the `publish-docker` `:latest` push step on the orb itself — the orb's own CI exercises its commands but not that specific combination of driver and tag operation. It was caught within minutes of the first user (lucos_contacts) hitting it, and fixed about 2.5 hours later via lucas42/lucos_deploy_orb#120.

Impact was limited to 1 repo because no other repo pushed to main during the broken window — a coincidence, not a mitigation.

### Stage 2: mirror overload during concurrent re-triggers (the evening fault)

#### Contributing factor: `docker.l42.eu` was provisioned undersized for concurrent streaming load

`lucos_docker_mirror` went live earlier today as a pull-through cache to replace a previous (decommissioned) GHCR build-context redirection approach. The orb change in lucas42/lucos_deploy_orb#118 made every publish job depend on it for both a login (`docker login docker.l42.eu`) and BuildKit pulls. The web front end is a small gunicorn app that proxies `/v2/` requests to an internal `registry:5000` distribution container; the registry streams blob responses through the gunicorn worker back to the CI client.

Under 8 simultaneous CI builds, each pulling node/python/php base images (tens of MB per layer), the proxy ran out of headroom:

- gunicorn workers were SIGKILL-ed with "Perhaps out of memory?" messages.
- Workers that didn't OOM hit the default 30s gunicorn timeout part-way through a blob transfer.
- In both cases the registry saw `write: broken pipe` after tens of megabytes already written — from the registry's point of view the client disappeared mid-stream.

This is a sizing and worker-model problem: gunicorn sync workers serialise one blob per worker. With a handful of workers and blobs that can legitimately take 30+ seconds each, the pool saturates instantly.

#### Contributing factor: orb has no fallback when the mirror is unavailable

Once the mirror was degraded, every CI build in the estate was forced to fail — there is currently no path in the orb to skip the mirror login or the `--config` that points BuildKit at it. A single shared infrastructure component has effectively become a hard dependency of every deploy. Tracked in lucas42/lucos_deploy_orb#122.

#### Contributing factor: downstream symptoms are visually confusing

The incident produced three different error signatures — `context deadline exceeded` on login, `manifest unknown` 404s, and BuildKit's `failed to compute cache key: unexpected commit digest ...: failed precondition` — all from the same underlying cause (workers dropping connections mid-transfer and the client receiving a truncated blob whose hash no longer matches the manifest). It initially looked like three separate bugs.

#### Contributing factor: avalon is already memory-pressured

Monitoring has been reporting `lucos_docker_health_memory_avalon` unhealthy for 240+ consecutive runs today ("High swap usage: 1255MB (threshold 1024MB)"). Deploying a new memory-hungry service onto a host that's already swapping makes OOM events more likely. This is a sysadmin concern and is cross-linked to the monitoring alert; not fixed in this incident.

---

## What Was Tried That Didn't Work

Everything tried worked on the first attempt:

- First pass assumption (before investigation) was that the 6 failed re-triggers were the same `docker tag` bug the morning's PR had fixed. One log pull per failing job quickly disproved that — the failed step names and error messages were clearly mirror-related, not `docker tag`-related.
- The instinct to reach for a config-as-code fix for the mirror was tempered by the fact that the service had been up and healthy until the very moment the parallel re-trigger wave hit. Restarting the web container was the minimal intervention and resolved the acute symptoms immediately, letting the two still-running pipelines complete.
- Re-triggers were done serially rather than in parallel on the way out. No further failures.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Replace the `docker tag` + `docker push` pattern in `publish-docker.yml` with `docker buildx imagetools create` (same fix that `publish-docker-multiplatform.yml` already had) | lucas42/lucos_deploy_orb#120 | Merged |
| Tune the gunicorn proxy (worker count, timeout, async workers, memory limit, saturation metrics) | lucas42/lucos_docker_mirror#19 | Closed (superseded by #22 / ADR-0002) |
| Replace Flask/gunicorn `web` with nginx proxy + `info` sidecar (ADR-0002 implementation — end-to-end streaming, no worker concurrency cap) | lucas42/lucos_docker_mirror#22 | Open |
| Make the mirror a best-effort optimisation — probe `docker.l42.eu/v2/` before login and skip the mirror config if it's unreachable, so a mirror outage doesn't hard-fail every estate deploy | lucas42/lucos_deploy_orb#122 | Open |
| Add a pre-publish test on the orb's own CI that exercises `publish-docker`'s `:latest` tag step end-to-end, to catch the next regression of this class | lucas42/lucos_deploy_orb#124 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
