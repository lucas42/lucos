# Incident: avalon load spike — cascading container healthcheck failures across estate

| Field | Value |
|---|---|
| **Date** | 2026-03-20 |
| **Duration** | ~22 minutes (00:05 UTC to 00:27 UTC) |
| **Severity** | Partial degradation — 28/31 services briefly erroring in monitoring; most services continued serving traffic throughout |
| **Services affected** | Estate-wide: 28/31 monitored systems showed errors at peak; most self-recovered within minutes as load dropped |
| **Detected by** | User report |

---

## Summary

A wave of simultaneous CI deployments to avalon caused host load to spike to ~40, saturating the server and causing Docker healthchecks to time out across many containers. Most containers recovered automatically as the load dropped. The monitoring container itself was restarted during the spike, causing it to briefly show 28/31 systems as erroring — a combination of genuine healthcheck failures and stale state from the restart. Full recovery to 31/31 healthy was achieved by 00:33 UTC through a combination of automatic recovery and manual CI retries.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~00:05 | Simultaneous CI deployments arrive on avalon (bulk secrets sweep had pushed commits to many repos on 2026-03-19; corresponding deploy jobs arriving together) |
| ~00:05–00:10 | Host load climbs to 28–40; `lucos_eolas_app` briefly consumes ~75% CPU during Django startup (`collectstatic` + arachne ingestor pulling `/metadata/all/data/` in a single 17-second request) |
| ~00:08 | CI builds for lucos_media_seinn, lucos_scenes, lucos_monitoring, lucos_media_metadata_api, lucos_media_manager (ceol), lucos_backups, and lucos_time all fail — their deploy jobs running during the load spike |
| ~00:10 | `lucos_arachne_web` exits (unhealthy); monitoring container restarts; `lucos_creds_configy_sync` SSH timeout logged |
| ~00:10 | Monitoring reports 28/31 systems erroring — mix of genuine failures and stale state from restart |
| ~00:11 | Host load begins dropping; most containers self-recover as healthchecks pass again |
| ~00:13 | Monitoring recovers to 22/31 erroring as poll cycle completes; `lucos_arachne_web` manually restarted to restore arachne service |
| ~00:13 | `lucos_contacts_web` and `lucos_eolas_web` port bindings confirmed restored (CI deploy succeeded) |
| ~00:19 | Host load at 0.58; batch CI retries triggered for 6 repos (seinn, scenes, monitoring, media_manager, backups, lucos_time) |
| ~00:24 | All 6 CI retries pass; `lucos_repos_app` restarted to trigger immediate audit sweep |
| ~00:26 | `lucos_creds_configy_sync` self-heals on regular schedule |
| ~00:27 | `lucos_media_metadata_api` CI retry triggered (missed in first retry batch) |
| 00:33 | `lucos_media_metadata_api` CI retry passes; monitoring reaches 31/31 healthy |

---

## Analysis

### Stage 1: Bulk CI deployment wave saturates the host

A bulk commit push on 2026-03-19 (adding `CODE_REVIEWER_APP_ID`/`CODE_REVIEWER_PRIVATE_KEY` to ~30 repos) had already caused one incident earlier that day (see lucas42/lucos#53). The resulting CI pipelines were staggered, but a second wave of deployments arrived simultaneously on avalon around 00:05 UTC, triggering multiple containers to restart and initialise concurrently.

During this window, `lucos_eolas_app` consumed ~75% CPU on startup: Django's `collectstatic` step runs on every container start, and the arachne ingestor immediately made a `GET /metadata/all/data/` request that took ~17 seconds and returned ~554KB. The combination of this with parallel restarts elsewhere pushed host load to 28–40 on a 7.6GB RAM server.

### Stage 2: Healthcheck cascade

With the host saturated, Docker healthchecks began timing out. Containers whose healthchecks failed were marked unhealthy; `lucos_arachne_web` exited entirely (despite `restart: always`, it had been up 9 hours and Docker's restart policy had cooled off). The monitoring container itself was restarted during the spike.

When monitoring restarted, it initialised with no cached state and began re-polling all 31 systems. Many systems that were actually still serving traffic (nginx was passing requests throughout) showed `fetch-info` failures — the monitoring container's DNS resolution was slow under load during its first poll cycle. This caused an inflated erroring count (28/31) that did not accurately reflect user-facing impact.

### Stage 3: CI builds fail during peak load

Seven CI deploy jobs ran during the 00:05–00:10 window. On avalon, `docker compose up --wait` waits for containers to pass their healthchecks; with healthchecks timing out under load, these jobs timed out and failed. This left the affected services with stale deployed versions (no functional regression — the containers were already running the same code) but triggered monitoring CI alerts that persisted until the builds were manually retried.

### Contributing factor: lucos_media_metadata_api missed in first retry batch

During the initial CI retry coordination, `media-api.l42.eu` was incorrectly mapped to `lucos_media_manager` rather than `lucos_media_metadata_api`. The CI check on `media-api.l42.eu` reports the CircleCI slug `gh/lucas42/lucos_media_metadata_api` — confirmed via the `/_info` endpoint. The error was caught and corrected before full resolution, but added ~10 minutes to the recovery time.

---

## What Was Tried That Didn't Work

### Initial diagnosis: network/DNS outage

The first hypothesis was a network or DNS outage, based on monitoring showing "DNS failure when trying to resolve IPv6 address" errors for nearly every service. This was incorrect — the errors were an artefact of the monitoring container restarting and performing its first poll cycle under a saturated host. Services were responding normally to nginx throughout. Confirmed by: `docker logs router` showing active traffic; direct `wget` from inside the monitoring container succeeding normally.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Investigate why `lucos_eolas_app` startup is CPU-intensive (Django collectstatic + arachne ingestor bulk fetch on every container start) | To be filed | Open |
| Consider rate-limiting or deferring arachne ingestor's `/metadata/all/data/` requests on startup | To be filed | Open |
| Add `lucos_media_metadata_api` to SRE agent memory mapping of `media-api.l42.eu` → repo name | (agent memory update) | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
