# Incident: lucos_media_metadata_manager login redirect loop after auth token URL fix

| Field | Value |
|---|---|
| **Date** | 2026-04-11 |
| **Duration** | ~25 minutes (14:27 UTC to 14:52 UTC) |
| **Severity** | Complete outage (login broken for affected users) |
| **Services affected** | media-metadata.l42.eu (lucos_media_metadata_manager) |
| **Detected by** | User report |

---

## Summary

PR lucas42/lucos_media_metadata_manager#208 (merged 14:27 UTC) fixed auth token URL exposure by adding a server-side 302 redirect to strip `?token=` from the URL after setting the cookie. This triggered a redirect loop for users who had legacy `auth_token` cookies scoped to a specific path (e.g. `/tracks/`) rather than `/`, because those old cookies took precedence over the freshly-set root-scoped cookie on the redirect follow-up, causing auth to fail on every attempt. Affected users were completely unable to log in. PR lucas42/lucos_media_metadata_manager#212 (merged 14:52 UTC) reverted the server-side redirect, replaced it with client-side `window.history.replaceState()`, and added expiry headers to clear any remaining legacy cookies.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 14:27:02 | PR lucas42/lucos_media_metadata_manager#208 merged; CI auto-deploy begins |
| ~14:28 | New container deployed with server-side redirect code; redirect loop begins for users with legacy path-scoped cookies |
| ~14:33 | First loop entries confirmed in production Apache logs |
| ~14:35 | User reports login broken |
| ~14:40 | SRE investigates; production logs pulled; root cause identified as cookie path conflict |
| ~14:45 | Hotfix PR lucas42/lucos_media_metadata_manager#212 raised |
| 14:52:32 | PR lucas42/lucos_media_metadata_manager#212 merged; CI auto-deploy begins |
| ~14:57 | New container deployed; login restored |

---

## Analysis

### Root cause: legacy path-scoped auth cookies conflicting with the new redirect

PR lucas42/lucos_media_metadata_manager#208 added this flow on successful authentication:

1. Validate token from `?token=` query parameter
2. Set `auth_token` cookie (scoped to `path=/`)
3. 302 redirect to the same URL without `?token=`

On the redirect follow-up request, the browser sends cookies. The problem: users who had authenticated before 2026-04-08 still had an old `auth_token` cookie in their browser scoped to a specific path (e.g. `/tracks/`), not `/`. This happened because the original `setcookie()` call (added in commit `c0c4079`, March 2023) had no `path` option — PHP's default is an empty string, so the browser computed the cookie path from the request URI directory, typically `/tracks/`.

When the PR #208 redirect fires, the browser has two cookies with the same name:
- `auth_token=<stale>; path=/tracks/` — the legacy cookie, path-specific
- `auth_token=<fresh>; path=/` — the newly-set root-scoped cookie

Per RFC 6265, cookies with a longer (more specific) path are sent first in the `Cookie:` header. The legacy cookie arrives first and PHP uses it. The stale token returns `401 Unauthorized` from `auth.l42.eu`, the server redirects back to `auth.l42.eu`, a new token is issued, and the cycle repeats — indefinitely.

### Why it wasn't caught

The redirect worked correctly in testing with a fresh browser profile, which had no legacy path-scoped cookie. No automated test exercised the cookie path conflict scenario. The stale cookie could only exist in a real browser session that had been open since before 2026-04-08 (when `bbb0b83` introduced the `path=/` option).

### Monitoring blind spot

The `/_info` endpoint is unauthenticated and continued returning 200, so monitoring showed the service as healthy throughout the incident. The authentication failure was entirely invisible to automated checks.

---

## What Was Tried That Didn't Work

Root cause was identified correctly on the first investigation pass. No dead ends.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Revert server-side redirect; expire legacy cookies; strip token client-side | lucas42/lucos_media_metadata_manager#212 | Done |
| Track the better client-side token strip approach as a follow-up to issue #170 | lucas42/lucos_media_metadata_manager#211 | Done (addressed in #212) |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.
