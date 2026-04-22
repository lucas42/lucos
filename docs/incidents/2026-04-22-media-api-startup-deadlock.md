# Incident: media-api startup deadlock causes ~30 min outage during Dependabot deploy wave

| Field | Value |
|---|---|
| **Date** | 2026-04-22 |
| **Duration** | ~30 minutes (08:03 UTC to 08:34 UTC) — with further intermittent drops until ~08:49 UTC |
| **Severity** | Partial degradation (single service complete outage, cascading to dependent services) |
| **Services affected** | `media-api.l42.eu` (complete outage), `media-metadata.l42.eu` (502 Bad Gateway via dependency), `schedule-tracker.l42.eu` (`lucos_arachne_ingestor_lucos_media_metadata_api` job failures) |
| **Detected by** | SRE ops check monitoring API review (observed after the fact — no human was paged) |

---

## Summary

During this morning's Dependabot auto-merge wave, a new deploy of `lucos_media_metadata_api` v1.0.14 failed to come up: the Go binary panicked with `unable to open database file: no such file or directory` despite the SQLite volume being mounted correctly. The previous container had already been removed, so `media-api.l42.eu` went cold for ~30 minutes. Service was restored by manually `docker start`ing the failed container — the same image + same config came up fine on the second attempt, confirming a transient startup condition rather than a code regression. A second failure mode surfaced later: the container repeatedly entered an unresponsive state post-startup (HTTP server accepted connections but never responded), causing CI redeploys to time out at `docker compose up --wait`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 07:07–07:32 | Dependabot auto-merge wave kicks off. 13 repos attempt to deploy. Many CI pipelines fail at `Populate known_hosts` step with `getaddrinfo creds.l42.eu: Temporary failure in name resolution` — CircleCI runner DNS blip. |
| ~08:03 | `lucos_media_metadata_api:1.0.14` container created on avalon. Go binary panics with `panic: unable to open database file: no such file or directory`. Exit code 2, zero log lines emitted before panic. |
| 08:03–08:33 | `media-api.l42.eu` returns HTTP 502 (nginx upstream unreachable). `media-metadata.l42.eu` 502 via the API dependency. `schedule-tracker.l42.eu` records `Ingest of lucos_media_metadata_api failed: 502 Server Error: Bad Gateway`. |
| 08:33 | SRE investigation begins — SSH to avalon, inspect container mount config. Confirms the volume mount spec is correct; the SQLite file (`/var/lib/docker/volumes/lucos_media_metadata_api_db/_data/media.sqlite`, 149 MB) is present and owned by 1001:1001 as expected. |
| 08:34 | `docker start 82064191ec29_lucos_media_metadata_api` succeeds. Same image, same mounts. Container logs `INFO Listening for incoming connections port=3002` and serves `/_info` normally. Service restored. |
| ~08:41 | Something (possibly a CI-retry-triggered `docker compose up`) recreates the container. It starts, logs `Listening`, but within minutes the HTTP server stops responding — `/_info` timeouts. |
| 08:42 | `lucos_monitoring` (freshly redeployed to v1.0.17 at 08:40) fires estate-wide alerts as it rebuilds state from scratch. False positives on TLS + fetch-info for ~20 services. Recoveries fire from 08:43. |
| 08:45–08:49 | Healthchecks on media-api repeatedly time out. `docker restart` restores service again. |
| ~08:49 | Media-api stable. Issue [`lucas42/lucos_media_metadata_api#184`](https://github.com/lucas42/lucos_media_metadata_api/issues/184) filed for the post-startup deadlock pattern. |

---

## Analysis

This was two distinct failures on the same container, interleaved with unrelated CI-side DNS noise.

### Failure 1: startup panic on volume that was correctly mounted

The new container's `docker inspect` showed both bind volumes (`lucos_media_metadata_api_db` → `/var/lib/media-metadata`, `lucos_media_metadata_api_exports` → `/var/lib/exports`) attached exactly as in the old container. The `media.sqlite` file was present in the volume throughout. Yet the Go binary panicked at init with `unable to open database file: no such file or directory` before emitting any log line.

A test run of the same 1.0.14 image with the same volume mounted (`docker run --rm -v lucos_media_metadata_api_db:/var/lib/media-metadata …`) succeeded at the DB-open step, panicking later on expected missing env vars. So the image and volume data are fine.

The most plausible explanation is a **container-init-time race** where runc's mount propagation did not complete before the binary's `main()` executed. This is rare but documented — typically surfaces when a volume driver is under load or when a prior container has not fully released a bind to the same volume. The old `lucos_media_metadata_api:1.0.13` container exited at ~07:58 (exit code 2, likely the orb's `docker compose down` step), and the new one was created ~5 minutes later. Overlap is possible if the orb issued `docker compose up` before the old tear-down fully unmounted.

Restarting the same container succeeded. No code or config change was needed.

### Failure 2: HTTP server enters stuck state shortly after startup

Separately, the new 1.0.14 container — once running — would serve requests for a short period, then enter a state where:
- The Go process is alive (PID 1, CPU ~1.4%, memory ~165 MiB — idle, not crashing).
- Port 3002 is listening (confirmed via `/proc/1/net/tcp`).
- Existing client connections are in `TCP_FIN_WAIT2` — accepted but never cleanly closed.
- New requests hang until the client times out; Docker healthcheck logs `Health check exceeded timeout (5s)` repeatedly.
- `docker restart` clears the state; the container comes back responsive in ~200 ms.

No panic, no stack trace, no error log — the HTTP handler is simply blocked. Candidate causes (tracked in [`lucas42/lucos_media_metadata_api#184`](https://github.com/lucas42/lucos_media_metadata_api/issues/184)):

1. **Synchronous eolas calls in request handlers.** PR #73 added an eolas-fetching data migration for `mentions`/`about`/`language` tags. Eolas itself was under deploy pressure this morning (CircleCI red, its own fetch-info flapping), so slow or hanging outbound HTTP calls are plausible.
2. **SQLite writer-lock contention.** The 149 MB DB has an uncheckpointed 13 MB WAL; long writer locks serialise readers. The `/_info` handler runs four aggregate queries — any one of them blocking on a writer would stall the healthcheck.
3. **Goroutine leak** exhausting a shared mutex.

### Stage: `lucos_dns_bind` unavailable for ~20 min during its own deploy

**Correction to an earlier version of this report:** the CI DNS failures were first filed here as a "CircleCI / GitHub Actions runner DNS blip". That dismissal was wrong.

Actual sequence:

| Time (UTC) | Event |
|---|---|
| 07:07:54 | `lucos_dns` pipeline #199 starts its first `build-deploy` workflow attempt. |
| 07:10:27 | `lucos/deploy-avalon` begins `Pull container(s) onto remote box` on avalon. |
| 07:31:23 | Pull finishes (21 min — slow, because the docker mirror was under simultaneous load from the Dependabot wave). |
| 07:31:26 | `Deploy using docker compose` starts. Old `lucos_dns_bind` is stopped. New container is **Created** at 07:31:35. |
| 07:31:35 → 07:51:28 | New `lucos_dns_bind` container is in `Created` but not `Running` state. The bind process inside the container does not actually start for **20 minutes**. Root cause TBD (see follow-up below). Authoritative DNS for `*.l42.eu` and `*.s.l42.eu` is **unavailable** during this window — any cache miss from an external resolver returns SERVFAIL. |
| 07:51:28 | Bind process starts, initialises in ~30 s. |
| 07:51:58 | `Deploy using docker compose` completes. |
| 07:54:17 → 07:56:50 | `Send deploy log to loganne` fails 6 consecutive times (likely negative DNS caching of the earlier failures still lingering, or loganne itself briefly reachable only via stale cache). The `deploySystem` event for `lucos_dns v1.0.11` is never emitted — which is why the Loganne feed had no record of the deploy when this report was first investigated. |
| 08:38:03 | Re-run of the `build-deploy` workflow succeeds; `deploySystem` event fires at 08:39:57. |

Other CI pipelines that ran deploys during the 07:31–07:51 window (lucos_eolas `Populate known_hosts` failed 07:36:33, lucos_notes `Deploy using docker compose` failed 07:38:37, lucos_scenes `Deploy using docker compose` failed 07:37:25) hit the same authoritative-DNS-unavailable condition, manifesting as `getaddrinfo: Temporary failure in name resolution` errors against various `*.l42.eu` hostnames.

**The short 60 s TTL on zone records makes this worse than it needs to be** — any CI runner whose cache expired during the 20 min window would fail, and the existing 5× retry envelope (`max_auto_reruns: 5, auto_rerun_delay: 30s` on `Populate known_hosts`) maxes out at ~2.5 min. This is already tracked as [`lucas42/lucos_dns#28`](https://github.com/lucas42/lucos_dns/issues/28) — add `$TTL 300` to zone templates. Today's 20 min outage is a strong argument for raising its priority.

The 20 min bind-startup delay itself is a separate, new issue and needs investigating — a normal bind restart completes in ~30 s. A new issue will be raised for that.

### Stage: estate-wide alert burst at 08:42 — correctly real, not a monitoring artefact

**Correction to an earlier version of this report:** the 08:42 alert burst was first described here as a self-inflicted consequence of `lucos_monitoring` restarting with empty state. That framing was wrong — `lucos_monitoring#87` already added a warm-up grace period (skip `state_change` when a host is seen for the first time, i.e. not yet in `SystemMap`), and the commit for that fix shipped on 2026-03-20. That fix is working: the alerts at 08:42:21–30 fired ~2 min after monitoring's deploy, well past the first poll cycle.

The alerts were therefore **real brief check failures**, not false positives. Seven services (scenes, configy, blog, notes, weightings, dns, monitoring) rolled out within a ~90-second window at 08:38:48–08:40:17 immediately before the burst. The most likely cause is the nginx router reloading mid-wave (front-side 502s for a few polling rounds) combined with avalon I/O load from the concurrent compose operations. No individual service was genuinely down — they recovered on the next poll — but the fetch-info checks did transiently fail, and monitoring was correct to alert.

No action needed on monitoring; the burst is a symptom of cramming seven deploys into 90 s on a single host, not a monitoring bug.

---

## What Was Tried That Didn't Work

- **Comparing docker-compose history for a volume-path change.** Assumed the panic was caused by a recent docker-compose.yml edit (new path, renamed volume). None found — the only recent docker-compose commit was the 2026-04-17 `${VERSION:-latest}` tag addition, unrelated to volumes.
- **Running the 1.0.14 image directly with `docker run`.** Panicked with the same message at first, but only because I forgot the volume mount. Once mounted correctly, it succeeded. This briefly misled the investigation into thinking the image itself was broken.
- **Searching for an existing issue on the 502 pattern.** None found. A wider search would have surfaced similar-shape container-startup issues in `lucos_deploy_orb` (`#21`, `#71`) for port contention but none covered volume-mount races.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Investigate and fix media-api post-startup HTTP deadlock | [`lucas42/lucos_media_metadata_api#184`](https://github.com/lucas42/lucos_media_metadata_api/issues/184) | Open |
| Add `$TTL 300` to zone templates so resolver caches cover deploy windows | [`lucas42/lucos_dns#28`](https://github.com/lucas42/lucos_dns/issues/28) | Open (pre-existing; today's incident is a strong argument for raising its priority) |
| Investigate why `lucos_dns_bind` took ~20 min to transition from `Created` to `Running` during today's deploy — normal restart is ~30 s | [`lucas42/lucos_dns#76`](https://github.com/lucas42/lucos_dns/issues/76) | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
