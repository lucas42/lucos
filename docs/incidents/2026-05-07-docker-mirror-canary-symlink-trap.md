# Incident: docker.l42.eu /_info hangs after canary metric reads /dev/stdout symlink from shared volume

| Field | Value |
|---|---|
| **Date** | 2026-05-07 |
| **Duration** | ~33 minutes (11:27 UTC to 12:00 UTC) |
| **Severity** | Partial degradation — monitoring alerts active, but mirror traffic continued to flow |
| **Services affected** | `lucos_docker_mirror_web` and `lucos_docker_mirror_info` (unhealthy); `schedule-tracker / lucos_docker_health_avalon` flapped |
| **Detected by** | Monitoring alert flagged inline by lucas42 (within minutes of the deploy) |

---

## Summary

A new `_metric_pull_rate()` canary in `lucos_docker_mirror_info` opened `/var/log/nginx/access.log` and iterated lines, expecting a real file written by nginx in the `web` sidecar via a shared `logs` named volume. The upstream `nginx:alpine` image ships that path as a symlink to `/dev/stdout`, and Docker's first-init copy of the volume preserved the symlink — so the info container's `for line in f:` ended up reading its own `/dev/stdout`, which never closes. Gunicorn workers timed out, were killed, restarted, took the next request, and looped. `/_info` stopped responding, healthchecks failed, and `lucas42/lucos_docker_health` reported the two containers as unhealthy. Mirror traffic continued unaffected. Resolved by reverting `lucas42/lucos_docker_mirror#57` via `lucas42/lucos_docker_mirror#58`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 11:27 | `lucas42/lucos_docker_mirror#57` ("Add docker_mirror_pull_count metric to /_info") merged to main |
| 11:27–11:30 | CI builds and deploys `lucas42/lucos_docker_mirror_web:1.0.21` and `lucas42/lucos_docker_mirror_info:1.0.21` to avalon. New named volume `logs` created and initialised from web image contents (which include the upstream nginx `access.log → /dev/stdout` symlink) |
| ~11:30 | Both `lucos_docker_mirror_web` and `lucos_docker_mirror_info` enter `unhealthy` state. Web continues to serve mirror traffic on `/v2/...` (200s); `/_info` hangs |
| ~11:32+ | `monitoring.l42.eu` starts alerting on `docker.l42.eu / fetch-info` (HTTP timeout). `lucas42/lucos_docker_health_avalon` cron starts reporting `Unhealthy containers: lucos_docker_mirror_web, lucos_docker_mirror_info` to `schedule-tracker` |
| ~11:42 | lucas42 spotted the alert on the monitoring dashboard, hypothesised PR #57 as cause, asked SRE to investigate |
| 11:43 | SRE investigation begins. Monitoring API confirms 2 failing systems, `docker.l42.eu` and `schedule-tracker.l42.eu` |
| ~11:46 | Container logs on avalon show the gunicorn worker timeout signature: `[CRITICAL] WORKER TIMEOUT` followed by Python traceback ending at `_metric_pull_rate()` `for line in f:`. SRE inspects the volume contents: `ls -la /var/log/nginx/` in the info container shows `access.log -> /dev/stdout` and `error.log -> /dev/stderr` |
| 11:50 | Root cause confirmed. Revert PR `lucas42/lucos_docker_mirror#58` opened |
| 11:56:05 | `lucos-code-reviewer` approves; auto-merge workflow lands the revert |
| 11:57 | CI begins building/deploying the reverted version (CircleCI pipeline 90, workflow `build-deploy`) |
| ~11:59 | Deploy completes. Web, info, and registry containers all transition to `healthy` on image `1.0.22`. `/_info` returns the expected payload with no `docker_mirror_pull_count` metric (correctly removed by the revert) |
| 12:00 | Monitoring API confirms `docker.l42.eu` recovered (`failing: 0`); `schedule-tracker / lucos_docker_health_avalon` already healthy. Verification window passes — no new alerts |

---

## Analysis

### Root cause: the `/dev/stdout` symlink survived volume init and resolved per-container

The upstream `nginx:alpine` image creates `/var/log/nginx/access.log` as a symlink to `/dev/stdout` and `/var/log/nginx/error.log` as a symlink to `/dev/stderr`. This is the standard nginx-on-Docker pattern — it gives `docker logs <container>` clean stdout/stderr capture without any logfile-rotation machinery.

PR #57 introduced two changes that interacted badly with this:

1. A new named volume `logs` mounted at `/var/log/nginx` in **both** the `web` container (rw) and the `info` sidecar (ro), so info could read what web wrote.
2. A new `_metric_pull_rate()` in `info/app.py` that ran on every `/_info` request, opening `/var/log/nginx/access.log` and iterating its lines.

When Docker first creates a named volume that's about to be mounted at a populated path in an image, it copies that path's image contents into the new volume (this is the "initialise from image" semantics that makes named volumes useful for Postgres-style data dirs). The two symlinks were copied in. Both containers, when they mount the volume, see the symlinks at the volume layer.

A symlink resolves at `open()` time **in the resolving process's namespace**. So:

- `nginx` in `web` opens `access.log`, the kernel resolves the symlink to `/dev/stdout`, which is `web`'s own stdout — which Docker captures as expected. `docker logs lucos_docker_mirror_web` keeps working.
- `_metric_pull_rate()` in `info` opens the same `access.log`, the kernel resolves the symlink to `/dev/stdout`, which is **info's** own stdout — a process-private stream that has no other writers and never closes.

`for line in f:` reads until EOF; on a stream that never closes, that's forever. Gunicorn's per-request timeout (default 30s) fires, `handle_abort` is called, `sys.exit(1)`, and the worker dies mid-request. A new worker is spawned, takes the next `/_info` request, hits the same trap. Web's healthcheck depends on info being healthy (compose `depends_on` + transitive healthcheck), so web also fails its healthcheck.

### Contributing factor: not testing locally

`docker compose up` would have surfaced the worker timeout in seconds — the bug is deterministic, not load-dependent. The "Test Locally Before Pushing" rule from SRE memory exists exactly for this class of failure (a previous 3-PR crash-loop incident on 2026-03-14 was caused by the same skipped step).

### Contributing factor: the bug is a known-pattern variant

The "named volume shadows image contents at mount path" pattern is in SRE memory (`pattern_named_volume_shadows_image.md`), with two prior incidents in 2026-03 (lucos_contacts and lucos_eolas serving stale static files for 5+ weeks). This case is the inverse direction — instead of stale image content surviving in a long-lived volume, fresh image content (the symlinks) was copied verbatim into a brand-new volume, where the per-container resolution semantics broke the metric. Both variants share the same lesson: **volume init is a snapshot, not a live mirror**, and any symlink in the snapshot is now a per-resolver liability.

### Why the impact was limited

`docker.l42.eu` continued to serve mirror traffic throughout (200s on `/v2/...`). The bug killed the `info` sidecar and the healthchecks, but nginx in `web` was unaffected — it just happened to share a parent compose with two sick children. CI builds across the estate that pull through the mirror would have been fine. The user-visible impact was confined to monitoring alerts and the deploy job for #57 itself failing (the `--wait` step in the deploy orb timed out waiting for the unhealthy containers).

---

## What Was Tried That Didn't Work

Nothing — the investigation went straight from "alert raised" to "root cause confirmed" without any false starts. SSH'ing into the info container, listing `/var/log/nginx/`, and seeing the two symlinks took under five minutes; the container logs corroborated the gunicorn timeout signature pointing at `_metric_pull_rate()`'s `for line in f:`.

The original PR's intent (a canary on `docker_mirror_pull_count` to detect silent fallback to Docker Hub) is sound; only the implementation of the log-reading was wrong. A fresh attempt with the fixes listed below is welcome.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Revert PR #57 to restore `/_info` health | lucas42/lucos_docker_mirror#58 | Merged 11:56Z, deployed 11:59Z, verified healthy 12:00Z |
| File follow-up issue with the four fix options for re-implementing the metric | lucas42/lucos_docker_mirror#59 | Open |
| Clean up the orphan `logs` volume left on avalon by #57's first-init (otherwise the symlinks persist for any future re-attempt) | lucas42/lucos_docker_mirror#59 (covered) | Pending |
| Verify monitoring API + `schedule-tracker / lucos_docker_health_avalon` recover post-deploy | — | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.
