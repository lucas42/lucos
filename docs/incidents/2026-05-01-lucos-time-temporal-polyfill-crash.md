# Incident: lucos_time crash-loop after non-Gregorian calendar feature deployed

| Field | Value |
|---|---|
| **Date** | 2026-05-01 |
| **Duration** | ~13 minutes (11:31 UTC to 11:44 UTC), with a brief secondary blip |
| **Severity** | Complete outage of `lucos_time`; partial degradation of `lucos_media_weightings` |
| **Services affected** | `lucos_time` (full outage), `lucos_media_weightings` (`time-api-reachable` failing) |
| **Detected by** | Monitoring alert (`fetch-info` on `am.l42.eu`) — 11:31:38 UTC |

---

## Summary

Two PRs merged in quick succession introduced non-Gregorian calendar matching: `lucas42/lucos_eolas#225` populated `temporal_id` for several calendars, and `lucas42/lucos_time#247` taught `/current-items` to evaluate them. Once both shipped to production, every `/current-items` request hit `@js-temporal/polyfill`'s Chinese-calendar code path, which threw `RangeError: Unexpected leap month suffix: Mo3` for today's date — taking the entire Node process down. Docker restarted it; the next request crashed it again. nginx served 502s for 13 minutes until a hotfix (`#251`) wrapped the per-calendar logic and the `/current-items` handler in try/catch.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 11:24:41 | `lucas42/lucos_eolas#225` merged — populates `temporal_id` for chinese, hebrew, indian, islamic, gregory |
| 11:26:29 | `lucas42/lucos_time#247` merged — adds non-Gregorian calendar matching to `/current-items` |
| 11:30:11 | `lucos_time v1.0.28` deployed to avalon |
| 11:31:38 | First monitoring alert: `lucos_time / fetch-info` failing |
| ~11:32 | `lucos_time` enters Docker restart loop; nginx serves 502s |
| 11:33:19 | Cascade alert: `lucos_media_weightings / time-api-reachable` failing (downstream consumer) |
| ~11:36 | `lucas42/lucos_time#250` filed by SRE with full root-cause diagnosis |
| 11:40:30 | Hotfix commit `2216d31a` (try/catch in `temporal-matcher.js` and `server.js`) pushed |
| 11:42:22 | Brief recovery — monitoring sees `lucos_time` healthy (likely a request that didn't hit the throw) |
| 11:42:45 | PR `#251` merged, closing `#250` |
| 11:42:46 | `#250` closed |
| 11:44:14 | Secondary fetch-info alert as `v1.0.29` deploy starts (deploy-window edge) |
| 11:44:44 | `lucos_time v1.0.29` deployed with the fix — service restored permanently |
| ~11:46 | Monitoring fully green |

---

## Analysis

### Latent polyfill bug exposed by data + code shipping together

The `@js-temporal/polyfill` Chinese-calendar helper has a known class of bug around leap-month suffix handling: its internal `monthCode` value (`"Mo3"`) fails its own re-validation, throwing `RangeError`. The bug was harmless until both halves of the feature went live:

- **Data side** (`lucas42/lucos_eolas#225`) populated `temporal_id` on five calendar entities including `chinese`, making them eligible for evaluation.
- **Code side** (`lucas42/lucos_time#247`) iterated every calendar with `temporal_id` set and called `withCalendar(temporalId).monthCode` on each.

Either change in isolation would have been safe. Together, they made `/current-items` evaluate the Chinese calendar on every request — and the polyfill throws on `2026-05-01`.

### No per-handler error boundary in the Node server

The bigger reliability gap was that `src/server.js`'s `/current-items` handler had no try/catch at the top level. A single uncaught `RangeError` from inside the handler tore down the entire HTTP server process. Docker restarted it, the next request hit the same code path, and the cycle repeated for 13 minutes.

A single bad downstream library should not be able to take a service into a hard restart loop. The hotfix correctly addressed both the immediate cause (skip the bad calendar) and the latent reliability gap (catch any future unhandled throw from a request handler).

### Detection was fast; the data/code coupling was the slow part

Monitoring alerted within ~80 seconds of the deploy completing, and SRE filed `#250` with a complete diagnosis (stack trace, root cause, proposed fix) within a few minutes of that. The slow part of the timeline was the data+code coupling: there was no integration test exercising every calendar with `temporal_id` populated, so the bug was not caught before merge.

---

## What Was Tried That Didn't Work

Nothing went wrong during the response itself — diagnosis was clean and the hotfix worked first time. There was a brief secondary fetch-info alert at 11:44:14 during the `v1.0.29` deploy window that initially looked like the fix hadn't worked, but it was just deploy-window noise — `v1.0.29` came up healthy 30 seconds later and stayed that way.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Hotfix shipped (try/catch in `temporal-matcher.js` and `server.js`, plus tests) | `lucas42/lucos_time#251` | Done (merged 11:42:45) |
| Consider an integration test that exercises every `temporal_id`-populated calendar against the current date in CI, so a polyfill-bad date is caught before merge | `lucas42/lucos_time#252` | Open — discussion |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
