# Incident: lucos_media_weightings webhooks 400-fail for ~75 min after SSRF guard rejects manager URLs

| Field | Value |
|---|---|
| **Date** | 2026-04-27 |
| **Duration** | ~1h15m (12:51 UTC to 14:06 UTC) |
| **Severity** | Partial degradation (single-service feature outage; no data loss; recoverable) |
| **Services affected** | `media-weighting.l42.eu/weight-track` (every Loganne `trackAdded` / `trackUpdated` webhook returned HTTP 400) |
| **Detected by** | User report (lucas42); also surfacing via Loganne `webhook-error-rate` check |
| **Source issue** | [`lucas42/lucos_media_weightings#195`](https://github.com/lucas42/lucos_media_weightings/issues/195) |

---

## Summary

`lucos_media_weightings` v1.0.28 introduced a Server-Side Request Forgery (SSRF) guard inside `fetchTrack` that rejected any URL not starting with the configured `MEDIA_API` origin (`https://media-api.l42.eu`). The webhook handler passes `event["url"]` (which lucos_media_metadata_api populates with `https://media-metadata.l42.eu/tracks/N` — the manager's resource URL, not the API URL) to `fetchTrack`. Every track-related webhook hit the guard, raised `ValueError`, and returned HTTP 400 to Loganne. A first attempt at fixing the issue ([`lucas42/lucos_media_weightings#196`](https://github.com/lucas42/lucos_media_weightings/pull/196)) added the manager origin as trusted and tried to follow the redirect to the API — but the manager doesn't redirect track URLs to the API; it redirects unauthenticated requests to `auth.l42.eu`, so the second-hop validation rejected the auth-redirect and the webhook still returned 400. The actual fix shipped in [`lucas42/lucos_media_metadata_manager#270`](https://github.com/lucas42/lucos_media_metadata_manager/pull/270): the manager now performs JSON content negotiation on track URLs and redirects `Accept: application/json` requests to the API (which `fetchTrack`'s redirect-handling code then follows successfully).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 12:51:09 | `lucos_media_weightings` v1.0.28 deployed to avalon. Includes [`lucas42/lucos_media_weightings#189`](https://github.com/lucas42/lucos_media_weightings/pull/189) ("Validate fetchTrack URL against configured media API to prevent SSRF") which adds the SSRF guard. Webhook outage begins. |
| 12:52:21 | v1.0.29 deployed (unrelated dependent change). |
| 12:56:09 | First observed `trackUpdated` event hits the bug. Loganne logs `https://media-weighting.l42.eu/weight-track: failure, errorMessage: "Server returned Bad Request"`. The `ceol.l42.eu` and `arachne.l42.eu` webhooks for the same event succeed — the bug is confined to media-weightings. Loganne `webhook-error-rate` check turns red. |
| ~13:25 | SRE investigates per user report. Confirms the failure mode — `fetchTrack(event["url"])` rejects `https://media-metadata.l42.eu/tracks/15061` because it doesn't start with `MEDIA_API` (`https://media-api.l42.eu`). Files [`lucas42/lucos_media_weightings#195`](https://github.com/lucas42/lucos_media_weightings/issues/195). |
| 13:36:11 | [`lucas42/lucos_media_weightings#196`](https://github.com/lucas42/lucos_media_weightings/pull/196) merged: adds manager origin to trusted list, makes `fetchTrack` follow one redirect manually with auth header re-sent. Built on the assumption that the manager would 302 to the API when given `Accept: application/json`. |
| 13:38:15 | v1.0.30 deployed with `lucas42/lucos_media_weightings#196`'s fix. Issue `lucas42/lucos_media_weightings#195` auto-closed by the merge. |
| ~13:43–13:45 | SRE retries the original failing event via Loganne's per-UUID retry API. Webhook **still returns 400**. Captures the actual production redirect with `curl`: `https://media-metadata.l42.eu/tracks/15061` returns 302 to `https://auth.l42.eu/authenticate?redirect_uri=…`, regardless of `Accept: application/json` and `Authorization: Bearer …`. The manager uses session-based auth via auth.l42.eu and doesn't honour bearer tokens at all. `lucas42/lucos_media_weightings#196`'s redirect-handling code therefore rejects the auth-service redirect target and raises `ValueError` → 400. |
| ~13:50 | SRE reopens `lucas42/lucos_media_weightings#195` with the trace and recommends fixing it on the manager side: add JSON content negotiation so `media-metadata.l42.eu/tracks/N` with `Accept: application/json` returns a 302 to the API instead of to the auth service. |
| 14:00:41 | [`lucas42/lucos_media_metadata_manager#270`](https://github.com/lucas42/lucos_media_metadata_manager/pull/270) merged: `conneg.php` now serves a 302 to the corresponding `media-api.l42.eu/v3/tracks/N` for any `Accept: application/json` request on a track URL. Manager redeploy follows. |
| ~14:06 | SRE confirms via webhook retry that `media-weighting.l42.eu/weight-track` now succeeds. Issue `lucas42/lucos_media_weightings#195` closed. Loganne `webhook-error-rate` clears. |

---

## Analysis

### Root cause: SSRF guard added without checking the actual shape of the URLs it was guarding

[`lucas42/lucos_media_weightings#189`](https://github.com/lucas42/lucos_media_weightings/pull/189) introduced two changes together:

1. The webhook handler started calling `track = fetchTrack(event["url"])` to re-fetch current state from the source, instead of trusting the inline `event["track"]` payload (idempotency improvement).
2. `fetchTrack` validated `url.startswith(apiurl + "/")` to prevent the bearer token from being forwarded to an attacker-controlled URL (defence in depth).

Both changes are individually reasonable. The combination assumed `event["url"]` would be on the API hostname — which it isn't. `lucos_media_metadata_api` populates `event["url"]` with the canonical resource URL on `media-metadata.l42.eu` (the manager UI), not with an API URL on `media-api.l42.eu`. This was a stable convention in the webhook payload long before `lucas42/lucos_media_weightings#189`, but neither the PR description nor the review surfaced it — both PR author and reviewer were focused on the SSRF correctness of the validator and didn't check what shape of URL the webhook handler actually receives.

### Contributing factor: `lucas42/lucos_media_weightings#196` was based on a false assumption never tested against production

[`lucas42/lucos_media_weightings#196`](https://github.com/lucas42/lucos_media_weightings/pull/196) fixed the `ValueError` symptom by adding the manager origin as a second trusted origin and following one redirect manually. This passes if you assume the manager redirects track URLs to the API.

It does not — and a one-line `curl -I -H 'Accept: application/json' -H 'Authorization: Bearer …' https://media-metadata.l42.eu/tracks/N` against production would have shown that immediately. The actual response is `HTTP/1.1 302 Found / Location: https://auth.l42.eu/authenticate?redirect_uri=…`, because the manager is a PHP UI with session-based auth that doesn't recognise bearer tokens. `lucas42/lucos_media_weightings#196`'s tests mocked the manager response shape to match the assumption rather than capturing the real production behaviour, so it shipped, deployed, and didn't actually fix anything.

This is the more interesting lesson from the incident. The PR was thoroughly designed and code-reviewed, but the review didn't catch that "the manager redirects to the API" was a load-bearing premise that hadn't been verified.

### Contributing factor: webhook handler didn't surface the failure mode in service logs

Production container logs during the outage showed `POST /weight-track` lines but no error detail — the `error()` call inside the `except Exception` handler logs the message, but the 400 path raises `ValueError` before reaching that handler. Anyone reading logs to triage the issue would see only "POST happened" and not "POST returned 400 because of X". The actual diagnosis came from Loganne's webhook delivery record, not from the service's own logs.

This isn't strictly a regression introduced today — the handler hasn't ever logged the `ValueError` path — but the SSRF guard is the first reason webhook handlers commonly hit `ValueError`, so it's the first time this gap mattered.

---

## What Was Tried That Didn't Work

- **`lucas42/lucos_media_weightings#196`'s "follow the redirect manually" approach.** Built on the unchecked assumption that the manager would 302 to the API. Deployed in v1.0.30 at 13:38; webhook still 400'd on retry. The fix was correct in shape (it would have worked if the manager behaved as assumed) but the assumption itself was the bug.
- **Initially looking at it as a media-weightings bug.** The first triage assumed the fix needed to land in the webhook handler — either by reverting `lucas42/lucos_media_weightings#189`'s `fetchTrack(event["url"])` change or by reconstructing the API URL from the track id (`fetchTrack(apiurl + "/v3/tracks/" + str(event["track"]["id"]))`). Both would have worked, but the chosen fix landed on the manager side instead — which is more consistent with the design intent that `event["url"]` should be a fetchable resource URL.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Manager: serve 302 to API for `Accept: application/json` on track URLs | [`lucas42/lucos_media_metadata_manager#270`](https://github.com/lucas42/lucos_media_metadata_manager/pull/270) | Done (merged 2026-04-27 14:00 UTC) |
| Add unit tests for fetchTrack SSRF guard and redirect handling | [`lucas42/lucos_media_weightings#197`](https://github.com/lucas42/lucos_media_weightings/issues/197) / [`PR #199`](https://github.com/lucas42/lucos_media_weightings/pull/199) | Done (merged 2026-04-27 14:05 UTC) |
| `fetchTrack`: SSRF guard bypassed by multi-hop redirects (the second `requests.get` uses `allow_redirects=True`, so further hops are unvalidated) | [`lucas42/lucos_media_weightings#201`](https://github.com/lucas42/lucos_media_weightings/issues/201) / addressed in [`PR #199`](https://github.com/lucas42/lucos_media_weightings/pull/199) (commit `3f45bee3`) | Done (merged 2026-04-27 14:05 UTC) |
| `fetchTrack`: missing `Location` header on a 302 raises `KeyError` instead of a clean `ValueError` | [`lucas42/lucos_media_weightings#202`](https://github.com/lucas42/lucos_media_weightings/issues/202) / addressed in [`PR #199`](https://github.com/lucas42/lucos_media_weightings/pull/199) (commit `3f45bee3`) | Done (merged 2026-04-27 14:05 UTC) |
| Tidy up test suite: consolidate test files, simplify CI command, update README | [`lucas42/lucos_media_weightings#198`](https://github.com/lucas42/lucos_media_weightings/issues/198) / [`PR #200`](https://github.com/lucas42/lucos_media_weightings/pull/200) | In progress |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

The bearer token used by `fetchTrack` was never sent to an untrusted host: the SSRF guard rejected the URL before any HTTP request was made, which is exactly the defence-in-depth behaviour `lucas42/lucos_media_weightings#189` was designed for. The bug that `lucas42/lucos_media_weightings#189` introduced was excessive blocking, not under-blocking.
