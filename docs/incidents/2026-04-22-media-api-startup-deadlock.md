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

### Stage: CircleCI runner DNS transient (unrelated, but coincident)

Separately, a cluster of CI failures at 07:07–07:32 resolved to a single symptom: the `Populate known_hosts` step failing with `getaddrinfo creds.l42.eu: Temporary failure in name resolution`. This is a CircleCI runner DNS resolver issue — DNS for `creds.l42.eu` resolves correctly from outside (1.1.1.1, 8.8.8.8, and the author's machine all return the correct A record), and our authoritative `lucos_dns_bind` had no restart in that window. The 6 auto-retries (`max_auto_reruns: 5, auto_rerun_delay: 30s`) all hit the same DNS failure because it persisted beyond the 2.5 min retry envelope.

This was not directly related to the media-api outage, but contributed to the overall CI red appearance this morning and delayed other deploys. Likely CircleCI-side only; no action on our side.

### Stage: monitoring restart caused self-inflicted alert burst

`lucos_monitoring` deployed a new version (v1.0.17) at 08:40. On startup, its in-memory state of "was this system healthy last time?" resets, so the first poll of each service generates a `monitoringAlert` event — even when the service is in fact healthy. This fired ~25 simultaneous alerts at 08:42, of which almost all emitted a matching `monitoringRecovery` at 08:43. Not harmful, but noisy in Loganne and confusing during triage. Worth a separate look at whether monitoring should persist or defer alerting during its own startup.

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
| Consider whether `lucos_monitoring` should defer alerting during its own startup to avoid false-positive alert bursts | To be raised | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
