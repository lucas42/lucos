# Incident: photos down for ~9 hours after a SQLAlchemy 2.1 bump changed the default Postgres driver

| Field | Value |
|---|---|
| **Date** | 2026-09-28 |
| **Duration** | ~9h14m (07:52Z first failed start → 17:05:55Z restored) |
| **Severity** | Complete outage (photos.l42.eu), plus silent failure of all background photo processing |
| **Services affected** | `lucos_photos` (api and worker) |
| **Detected by** | Monitoring: `lucos_photos` `fetch-info` alerted at 07:48:26Z and stayed red. Acted on only when lucas42 asked about a "flappy" `lucos_docker_health` alert at ~16:50Z |

---

## Summary

At 07:20–07:27Z Dependabot merged `sqlalchemy>=2.1.0` into `lucos_photos`' api, worker and shared package (lucas42/lucos_photos#541, lucas42/lucos_photos#542, lucas42/lucos_photos#544). SQLAlchemy 2.1 resolves a bare `postgresql` URL to the **psycopg 3** dialect, but the images only ship `psycopg2-binary`. From the first deploy onwards, every database connection raised `ModuleNotFoundError: No module named 'psycopg'`. The api crash-looped in its startup migration (500+ restarts), so photos.l42.eu was down. The worker failed every database touch while its healthcheck stayed green.

Monitoring caught it within a minute and the deploy job went red, but nothing acted on either signal for about nine hours. The fix, lucas42/lucos_photos#547, is a one-line change that names the driver explicitly (`postgresql+psycopg2`). Its first deploy then failed to pull an image through the newly enabled registry mirror (lucas42/lucos#307). A re-run succeeded, and service was restored at **17:05:55Z**.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 07:19–07:27 | Dependabot merges six PRs into `lucos_photos` (lucas42/lucos_photos#540 to lucas42/lucos_photos#545). Three of them bump SQLAlchemy to `>=2.1.0`. Pre-merge CI passes on all of them, because every test engine is SQLite. |
| 07:47:55 | First post-merge deploy (pipeline 1006, the shared-package bump) starts `lucos/deploy-avalon`. |
| 07:48:26 | **`lucos_photos` `fetch-info` alerts** (`HTTP Request timed out`). It never recovers. |
| 07:52:16 | The worker starts on 1.0.173. It reports healthy from here on while failing every database call. |
| 07:53:18 | `deploy-avalon` fails. The four later pipelines (1007, 1008, 1009, 1012) also fail at `deploy-avalon`, the last at 08:09:27. `lucos_photos`' `circleci` check goes red (alert at 08:18:34). |
| 07:54 onwards | `lucos_docker_health`'s `avalon` check reports `lucos_photos_api` crash-looping, but **alternates alert and recovery about every 30 minutes**. |
| ~16:45 | lucas42 asks for the "flappy" `lucos_docker_health` alert to be investigated. |
| 16:50 | SRE finds photos_api on 1.0.173 with 515 restarts, `ModuleNotFoundError: No module named 'psycopg'`, and `photos.l42.eu/_info` timing out. |
| ~16:52 | Root cause confirmed from the traceback (`dialects/postgresql/psycopg.py` → `import psycopg`). The worker is found failing identically (~240 tracebacks an hour). |
| 16:53 | Fix verified locally: the fixed engine resolves to `psycopg2` under SQLAlchemy 2.1.1, a bare-`postgresql` control reproduces the production error, and a throwaway pgvector:pg16 runs all 11 migrations with `/_info` 200 and `db-reachable: true`. lucas42/lucos_photos#547 opened. |
| 16:56:54 | lucas42/lucos_photos#547 approved by lucos-code-reviewer and merged (main pipeline 1019). |
| 17:00:05–17:00:45 | The fix's `deploy-avalon` fails at "Pull container(s) onto remote box": `docker.io/pgvector/pgvector:pg16: pull access denied … no basic auth credentials`. This is the first photos deploy since the ~16:29Z `systemctl reload docker` that enabled the `docker.l42.eu` mirror (lucas42/lucos#307). |
| ~17:03 | Manual pulls of the same images on avalon succeed, falling back to Docker Hub. SRE re-runs the workflow from the failed job. |
| **17:05:55** | **Restored.** api and worker on 1.0.174; `photos.l42.eu/_info` 200 with `db-reachable: true`, `redis-reachable: true`; 0 restarts. |
| 17:06:52 | Verified: the worker completes 2 real sweeps with 0 errors, and monitoring shows `lucos_photos` and `lucos_docker_health` healthy. |

---

## Analysis

### Stage 1: a minor-version bump changed a default

SQLAlchemy 2.1 changed which DBAPI a bare `postgresql://` URL loads: psycopg 3 instead of psycopg2. `shared/lucos_photos_common/database.py` built its URL with `drivername="postgresql"`, relying on that default, while every requirements file pins `psycopg2-binary`. Nothing in the code was wrong under 2.0. The change that broke it was a Dependabot requirement bump.

### Stage 2: CI could not see it

Every test engine in `lucos_photos` is SQLite (`api/tests/conftest.py`, `worker/tests/conftest.py`, `worker/tests/test_sweep.py`), so no test ever imports a Postgres DBAPI. All six Dependabot PRs passed pre-merge CI and auto-merged. The first thing to exercise the real driver was the production container starting.

### Stage 3: detection worked; response didn't

This isn't a monitoring gap. `lucos_photos` alerted at 07:48:26Z, a minute after the first deploy began, and stayed red. `deploy-avalon` failed on all five post-merge pipelines, and the `circleci` check went red too. Both are the right signals, delivered at the right time. The outage lasted nine hours because nothing acted on them. This is the alert-to-action gap tracked in lucas42/lucos#290.

### Stage 4: the one signal that was acted on was the noisy one

What finally prompted investigation was `lucos_docker_health` looking **flappy**. Its `avalon` job reported the crash-loop on 524 runs but reported success on 20. The monitoring check passes if *any* of the last 5 runs succeeded, so each lucky run produced a recovery, and five failures later a new alert. A continuous nine-hour outage showed up as about 17 short alert/recovery cycles. That's the pattern lucas42/lucos_docker_health#108 fixed for the stuck-starting case. Its "RestartCount rising" check is visibly working here, but some polls still see no rise. My unverified guess is that restart backoff lengthens to around the 60s poll interval, so some polls land between restarts.

A flapping alert invites "it's recovering on its own", and it's the only reason this was looked at when it was. It's worth being clear that flapping didn't help here: the correctly-red `lucos_photos` signal had already been there all day.

### Stage 5: the fix's deploy hit a separate, same-day regression

The first deploy of lucas42/lucos_photos#547 failed at the image pull. Since lucas42's docker reload at ~16:29Z enabled `registry-mirrors: ["https://docker.l42.eu"]` (lucas42/lucos#307), the mirror has been returning **401** to avalon's daemon for some images: `pgvector/pgvector` manifests, `lucos_photos_api` blobs and `hello-world`, though `library/python`, `node` and `lucos_navbar` got 200, per the router log. A manual `docker pull` falls back to Docker Hub and succeeds. The deploy's `docker compose pull` at 17:00 didn't fall back. Why the two paths differ isn't yet established; it's tracked on the reopened lucas42/lucos#307. A re-run succeeded, so the mirror cost about 5 minutes here. But any deploy on avalon is currently exposed.

---

## What Was Tried That Didn't Work

Nothing during the fix. The traceback named the missing module directly, and the local build confirmed the mechanism before the PR went up.

One thing initially looked relevant and wasn't: the investigation started as a check on whether the previous day's avalon changes (the DNS revert in lucas42/lucos_media_seinn#639, the container restarts, and the `systemctl reload docker` for lucas42/lucos#307) caused the docker_health flapping. They didn't. The crash-loop is entirely the SQLAlchemy bump.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Pin the driver explicitly (`postgresql+psycopg2`) | lucas42/lucos_photos#547 | Done (deployed 17:05Z) |
| CI must fail when a built image can't start and reach Postgres with the real driver | lucas42/lucos_photos#548 | Open |
| docker_health crash-loop detection: one flat `RestartCount` poll resets the streak, so a long crash-loop still reports success ~1 run in 27 | lucas42/lucos_docker_health#122 | Open |
| The `docker.l42.eu` mirror returns 401 to avalon's daemon for some images, and deploy pulls don't fall back | lucas42/lucos#307 (reopened) | Open |
| Alerts that stay red for hours aren't acted on | lucas42/lucos#290 | Existing; this incident is a data point |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
