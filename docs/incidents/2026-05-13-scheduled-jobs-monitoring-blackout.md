# Incident: scheduled-job monitoring blackout after ADR-0004 flag-day cutover

| Field | Value |
|---|---|
| **Date** | 2026-05-13 |
| **Duration** | ~7 hours so far (15:40 UTC start; recovery TBD pending production cred + PR #233 merge) |
| **Severity** | Partial degradation — monitoring blind to every scheduled-job failure across the estate |
| **Services affected** | `lucos_monitoring` (data path); every system with a scheduled-job check (detection path) |
| **Detected by** | User report from lucas42 |

Source issue: [`lucas42/lucos_monitoring#234`](https://github.com/lucas42/lucos_monitoring/issues/234)

---

## Summary

Today's roll-out of [ADR-0004](https://github.com/lucas42/lucos/blob/main/docs/adr/0004-scheduled-jobs-monitoring-architecture.md) relocated scheduled-job checks from `lucos_schedule_tracker`'s `/_info` payload onto a new `GET /jobs` endpoint, polled by a new monitoring fetcher (`fetcher_scheduled_jobs`) which attributes each check to its owning system. The flag-day cutover ([`lucas42/lucos_schedule_tracker#83`](https://github.com/lucas42/lucos_schedule_tracker/pull/83)) shipped at the same time as the new fetcher ([`lucas42/lucos_monitoring#231`](https://github.com/lucas42/lucos_monitoring/pull/231)) — but the new fetcher's required environment variable `SCHEDULE_TRACKER_ENDPOINT` was never added to monitoring's `docker-compose.yml` `environment:` whitelist, nor to lucos_creds. Inside the running container the variable was the empty string, so the fetcher's HTTP request had no scheme; `httpc:request` rejected it with `{no_scheme}` once per minute, and every scheduled-job check vanished from monitoring at the moment of the cutover. Loganne event recording remained healthy; only monitoring-side detection was offline.

Resolution is [`lucas42/lucos_monitoring#233`](https://github.com/lucas42/lucos_monitoring/pull/233) (adds the variable to the compose passthrough *and* drops the `++ "/jobs"` path-append in the fetcher so the env var is used verbatim, matching the lucos `_ENDPOINT`/`_ORIGIN` convention) plus a matching `SCHEDULE_TRACKER_ENDPOINT=https://schedule-tracker.l42.eu/jobs` write to `lucos_monitoring/production/.env` in lucos_creds (lucas42-only, as agents are read-only on creds). Recovery verification TBD pending both landing and the next monitoring deploy.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~15:04 | [`lucas42/lucos_schedule_tracker#82`](https://github.com/lucas42/lucos_schedule_tracker/pull/82) merges — `POST /v2/report-status` added (backwards-compatible) |
| ~15:14 | [`lucas42/lucos_schedule_tracker#83`](https://github.com/lucas42/lucos_schedule_tracker/pull/83) merges — `GET /jobs` added; `/_info` no longer emits synthetic scheduled-job checks (flag-day cutover) |
| ~15:39 | [`lucas42/lucos_monitoring#231`](https://github.com/lucas42/lucos_monitoring/pull/231) merges — `fetcher_scheduled_jobs` added, polling `${SCHEDULE_TRACKER_ENDPOINT}/jobs` every 60s |
| ~15:40 | `lucos_monitoring` redeploys to avalon with the new fetcher. `SCHEDULE_TRACKER_ENDPOINT` is unset inside the container; fetcher logs `transport error contacting schedule_tracker: {no_scheme}` and casts no updates. **Every scheduled-job check disappears from monitoring at this moment.** No `monitoringAlert` event fires for the disappearance because checks merely cease to exist — there is no failing state to alert on |
| ~21:55 | lucas42 notices scheduled jobs are no longer showing on monitoring and reports to team-lead |
| ~22:01 | team-lead dispatches lucos-site-reliability |
| 22:03–22:12 | Investigation: confirmed `/jobs` serves 200 with 37 v1-shaped rows; confirmed `monitoring/_info` is post-deploy; cross-checked fetcher source and `monitoring_state_server` cast path (both would handle v1 synthetic IDs correctly via `{SystemStr, system}` fallback); confirmed `/api/status` lists 51 systems with **0 `scheduled-job` checks**; `docker exec lucos_monitoring printenv SCHEDULE_TRACKER_ENDPOINT` returns empty; container logs show `{no_scheme}` every minute |
| 22:13 | [`lucas42/lucos_monitoring#233`](https://github.com/lucas42/lucos_monitoring/pull/233) opened with the one-line compose fix |
| 22:21 | [`lucas42/lucos_monitoring#234`](https://github.com/lucas42/lucos_monitoring/issues/234) filed as the incident tracker; PR #233 updated with `Closes #234` |
| TBD | Production cred `SCHEDULE_TRACKER_ENDPOINT=https://schedule-tracker.l42.eu/jobs` added to `lucos_monitoring/production/.env` in lucos_creds — **TBD pending lucas42** |
| TBD | PR #233 reviewed and merged — **TBD** |
| TBD | `lucos_monitoring` redeploys; fetcher_scheduled_jobs receives the env var and starts casting scheduled-job checks — **TBD pending deploy** |
| TBD | Verification: `/api/status` contains `scheduled-job` checks; `{no_scheme}` warnings stop — **TBD pending verification** |

---

## Analysis

### Stage 1 — flag-day cutover with no end-to-end verification

ADR-0004's roll-out plan correctly identified that the new fetcher needed to work before the `/_info` cutover could safely happen. In practice the two PRs were merged within 25 minutes of each other and deployed on the same flag-day with no human-in-the-loop verification step between them. There was no point in the rollout where someone fetched `monitoring/api/status` and confirmed `scheduled-job` checks were present before pushing the `/_info` cutover. Had this check existed (manual or automated), the gap would have been caught within the first minute of the new fetcher's lifecycle.

### Stage 2 — code-vs-config drift on a new env var

The new fetcher reads `SCHEDULE_TRACKER_ENDPOINT` via `os:getenv("SCHEDULE_TRACKER_ENDPOINT", "")` (`src/fetcher_scheduled_jobs.erl` line 38). Adding a new env var to a service requires three coordinated edits:

1. The application code reads it.
2. `docker-compose.yml`'s `environment:` block forwards it from the host into the container.
3. The lucos_creds `.env` for each environment holds the value.

PR #231 made the first change but not the other two. The current code review process and CI do not catch this class of drift — there is no convention check that says "for every `os:getenv` of a non-default-fallback variable, the variable must appear in `docker-compose.yml`'s `environment:` block." The result was a silently-broken deploy.

A second latent convention mismatch came to light while writing the fix: the variable was named `SCHEDULE_TRACKER_ENDPOINT`, but the code in PR #231 hard-coded a `++ "/jobs"` path append, treating the value as an origin. Per lucos convention, `_ENDPOINT`-suffixed variables hold the full URL (including path); `_ORIGIN` variables hold just the origin. The fix shipped in #233 moves the path into the env var value to match the convention without renaming; the matching cred is now `SCHEDULE_TRACKER_ENDPOINT=https://schedule-tracker.l42.eu/jobs`. The mismatch was masked by the primary bug (no cred ever reached the container), and would have surfaced as a second deploy-time failure had the original cred been written as an origin to match the old code's expectation.

### Stage 3 — empty default masking the failure mode

`os:getenv("SCHEDULE_TRACKER_ENDPOINT", "")` defaults to the empty string when the variable is unset. The fetcher then concatenates `"" ++ "/jobs"` → `"/jobs"`, which is a *valid string* but not a valid URL. `httpc:request` returns `{error, no_scheme}` rather than crashing, so the supervisor never restarts the fetcher and there is no startup-time error to surface in deploy logs. The defensive default — sensible in isolation — converted a fatal misconfiguration into a quiet 60-second-cycle warning that no monitoring rule was looking for.

### Stage 4 — disappearance is not detectable by existing alerts

Monitoring's alerting model is built around checks changing state from healthy to unhealthy. When a check *ceases to exist altogether* — as happened when both `/_info` stopped emitting scheduled-job checks and the new fetcher failed to start emitting them — no alert fires. The estate's monitoring blind spot was therefore invisible to the monitoring system that was supposed to detect it. The only signal was lucas42 visually noticing that the scheduled-jobs section had emptied out, ~6 hours after it happened.

---

## What Was Tried That Didn't Work

Diagnostic path was clean — no dead ends. Initial hypothesis space (from team-lead's brief) included five candidates: fetcher not running, env var missing, `/jobs` not deployed, fallback failing on synthetic IDs, attribution silently dropping rows. The first two diagnostic calls (curl `/jobs` to confirm schedule_tracker side, then `docker exec printenv` on the monitoring container) eliminated three hypotheses and confirmed the second within ~9 minutes.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Apply compose fix and add production cred to restore service | [`lucas42/lucos_monitoring#233`](https://github.com/lucas42/lucos_monitoring/pull/233) | In progress |
| Convention check: every `os:getenv` (or equivalent) of a non-fallback-defaulted env var must have the variable declared in `docker-compose.yml`'s `environment:` block | [`lucas42/lucos_repos#387`](https://github.com/lucas42/lucos_repos/issues/387) | Open |
| `SYSTEM` env var also missing from monitoring's compose passthrough — all three fetchers send empty User-Agent headers (ADR-0001 violation, pre-dates this incident) | [`lucas42/lucos_monitoring#235`](https://github.com/lucas42/lucos_monitoring/issues/235) | Open |
| Consider whether monitoring should alert on "check disappearance" (a check that existed in the previous poll but is absent now) in addition to healthy→failing transitions | [`lucas42/lucos_monitoring#236`](https://github.com/lucas42/lucos_monitoring/issues/236) | Open |

---

## Sensitive Findings

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.
