# Incident: Estate-wide rollout exhausts GitHub GraphQL rate limit, blocking 24 deploys

| Field | Value |
|---|---|
| **Date** | 2026-04-16 |
| **Duration** | ~90 minutes (01:05 UTC to ~02:35 UTC) |
| **Severity** | Partial degradation |
| **Services affected** | 24 services with blocked deploys (no services went down) |
| **Detected by** | Monitoring alerts (CircleCI check failures across 24 systems) |

Source issue: lucas42/lucos_deploy_orb#82

---

## Summary

An estate-wide Dockerfile migration (adding `ARG VERSION` / `ENV VERSION`) merged ~35 PRs within a ~10 minute window. Each merge triggered a CI pipeline that now runs `semantic-release` via the `calc-version` step, which uses the GitHub GraphQL API to analyse commits and create releases. The concurrent API calls exhausted the GraphQL rate limit for the CI token, causing the `Run Semantic Release` step to fail in 24 repos. Deploys were blocked but no running services went down — previous containers continued serving traffic. Manual re-runs after the rate limit reset restored all pipelines.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 00:14 | First `dockerfile-version-arg` branch pipelines triggered across ~35 repos |
| 01:05 | PRs begin merging to `main`, triggering `build-amd64` / `build-multiplatform` pipelines with `calc-version` |
| 01:06–01:16 | ~35 pipelines run `semantic-release` concurrently; GitHub GraphQL rate limit exhausted |
| 01:16 | First `RATE_LIMIT` / `graphql_rate_limit` errors observed in CI logs |
| 01:23 | Monitoring alerts fire — 24 CircleCI check failures visible on dashboard |
| 02:35 | SRE investigation identifies root cause, confirms rate limit has reset |
| 02:36 | All 24 failed workflows re-run from failed step via CircleCI API |
| 02:40 | 23/24 re-runs succeed; `lucos_media_import` fails on a transient Docker content lease error on xwing (unrelated) |
| 02:42 | `lucos_media_import` re-run succeeds on second attempt |
| 02:42 | `lucos_time` restarted on avalon to clear stale eolas cache (failed on startup during the deploy wave) |
| 02:45 | 49/50 monitoring checks healthy; only Loganne `webhook-error-rate` remains (self-clearing) |

---

## Analysis

### Rate limit exhaustion from concurrent `semantic-release` runs

The recently added `calc-version` step in `build-amd64` and `build-multiplatform` (lucas42/lucos_deploy_orb#73) runs `semantic-release` with the `@semantic-release/github` plugin. Each invocation makes multiple GitHub GraphQL API calls to analyse commit history, check existing tags, create a git tag, and publish a GitHub Release.

The GitHub GraphQL API has a point-based rate limit (5000 points/hour). With ~35 repos all running `semantic-release` simultaneously, the cumulative API calls exceeded the budget for the CI token (user ID 428847). The plugin's "success" step (creating the GitHub Release) was the specific failure point.

The `calc-version` command has no rate limit awareness — no retry logic, no backoff, no detection of rate limit headers. A single failure aborts the entire job.

### Estate-wide rollout pattern

The Dockerfile migration was rolled out as ~35 PRs that auto-merged in rapid succession. This is the expected pattern for estate-wide rollouts (dependabot batches, convention compliance fixes), but the new `calc-version` step in every build job means each merge now generates significantly more GitHub API traffic than before.

### Secondary effects

Two minor cascading issues occurred during the deploy wave:

1. **`lucos_time` eolas cache failure**: lucos_time restarted while lucos_eolas was also mid-restart. The eolas RDF cache refresh on startup got a 500 and doesn't retry automatically — required a manual container restart.

2. **Loganne webhook delivery failures**: 6 `albumCreated` events failed delivery to `arachne.l42.eu/webhook` during the deploy window when arachne was briefly cycling. Self-clearing as events age out of the 10K event memory window.

---

## What Was Tried That Didn't Work

Everything tried during this incident worked. The rate limit had already reset by the time investigation began, so the re-runs succeeded on the first attempt (except `lucos_media_import` which had a separate transient Docker error on xwing, resolved on second re-run).

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add retry-with-backoff to `calc-version` for rate limit errors | lucas42/lucos_deploy_orb#82 | Open |
| Implement dependabot update grouping to reduce daily PR volume | lucas42/lucos_repos#327 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
