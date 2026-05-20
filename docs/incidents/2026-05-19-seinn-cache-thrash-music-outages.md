# Incident: seinn service-worker cache thrash silently breaks music playback

| Field | Value |
|---|---|
| **Date** | 2026-05-19 (recurrence 2026-05-20) |
| **Duration** | ~15–30 min per occurrence (see Timeline) |
| **Severity** | Partial degradation (music playback only, internal-facing) |
| **Services affected** | lucos_media_seinn (web player), and as a cascading side effect: lucos_loganne (webhook-error-rate alerts) |
| **Detected by** | User report ("music isn't playing") on 2026-05-20; in retrospect the same failure mode produced loganne `webhook-error-rate` alerts on 2026-05-19 and earlier dates |

---

## Summary

The `lucos_media_seinn` web player, when run in a single long-lived browser tab over multiple hours, can enter a degraded state in which its service-worker `CacheStorage` thrashes — every freshly preloaded track is evicted before it can be played. The audio element fails, seinn reports the track as errored to media-manager, media-manager skips to the next track, and the cycle repeats once per few seconds. From the user's perspective, music silently stops playing. This happened on 2026-05-19 (~17:08–17:25 UTC) and again on 2026-05-20 (~10:55–11:12 UTC). On both occasions resolution was a manual `deviceSwitch` away from seinn (to "Phone" yesterday, "Galactica" today). The underlying trigger is a race condition in seinn's LRU bookkeeping ([`lucas42/lucos_media_seinn#456`](https://github.com/lucas42/lucos_media_seinn/issues/456)) that gradually scrambles the eviction ordering once the cache hits its 750 MB budget.

---

## Timeline

### 2026-05-19 occurrence

| Time (UTC) | Event |
|---|---|
| 17:08:05 | Last successful `trackUpdated ... finished playing` event (Cherry Tree Lane Part 1) before degradation. |
| 17:09:00 | loganne `monitoringAlert`: `webhook-error-rate` failing. Brief — recovers at 17:09:13. |
| 17:09:13 → 17:25:40 | Sustained burst of `trackUpdated ... errored` events from `lucos_media_metadata_api`. Two further `webhook-error-rate` alerts at 17:16:30 and 17:22:44 (loganne's outbound webhooks to `arachne.l42.eu`, `media-weighting.l42.eu`, and `ceol.l42.eu` failing with `fetch failed`). |
| 17:25:40 | `deviceSwitch` from `lucos_media_manager`: "Moving music to play on Phone". |
| 17:27:54 onward | Successful `trackUpdated ... finished playing` and `trackWeightingUpdated` events resume — playback restored on the new device. |

### 2026-05-20 occurrence

| Time (UTC) | Event |
|---|---|
| 10:55:46 | First `Webhook failure ... fetch failed` recorded in loganne's container logs. |
| 10:58 → 11:11 | Sustained burst of `trackUpdated ... errored` events, peaking at ~15/min. 33 events end up with permanently failed webhook deliveries (auto-retry-once exhausted). |
| 11:12:05 | loganne `monitoringAlertSuppressed` for `webhook-error-rate` — deploy-window suppression kicks in just before the `lucos_media_manager` deploy. |
| 11:12:13 | `deviceSwitch`: "Playing music on first device connected". |
| 11:12:14 | `deviceSwitch`: "Moving music to play on Galactica". |
| 11:12:25 | Unrelated coincident deploy: `lucos_media_manager v1.0.31` (PR `lucas42/lucos_media_manager#263` — parse JSON error payloads). Not the fix for this incident; an in-flight improvement to error visibility. |
| 11:13:22 | loganne `monitoringAlert` re-fires (deploy-window suppression ended). 33 stranded webhook events still in `status: failure`. |
| 11:15:49 | `trackUpdated ... finished playing` ("I'm From Further North Than You") — first track to actually complete after the deviceSwitch. Recovery confirmed. |
| ~11:18 | `lucos-site-reliability` runs `POST /events/retry-webhooks` on loganne. All 33 stranded webhook deliveries succeed on first retry. |
| 11:17:06 | loganne `monitoringRecovery` — all checks healthy. |

---

## Analysis

### Triggering condition: cache thrash in seinn's service worker

`lucos_media_seinn` caches preloaded audio in a CacheStorage cache (`tracks-v1`) capped at 750 MB by the LRU eviction logic added in [`lucas42/lucos_media_seinn#179`](https://github.com/lucas42/lucos_media_seinn/issues/179). Once the cache reaches that cap (achievable in a few hours of normal play — at ~10 MB per track, ~75 tracks is enough), every newly preloaded track triggers an eviction pass to make room.

The browser console log captured from the affected tab on 2026-05-20 showed:

| Pattern | Count |
|---|---|
| `cache-eviction.js:179 Cache eviction: removed <mp3 url>` | 158 |
| `index.js:50 Request not in cache <mp3 url>` | 187 |
| `preload.js:57 Failed to cache track: Cache.put() encountered a network error` | 1 |
| `poll.js:29 Error whilst polling — TypeError: Failed to fetch` | 5 (correlate to the brief media_manager deploy window) |
| `[Violation] Added non-passive event listener` | 450 (Chrome DevTools noise, unrelated) |
| Other | 0 |

The recurring eviction-of-same-URLs pattern (e.g. "Waterflame – Toy Soldiers", "Lindsey Stirling – Zi-Zi's Journey" appearing dozens of times each) shows the LRU ordering had diverged from reality — tracks were being evicted, requested again, fetched, cached, and immediately re-evicted, while older tracks from earlier in the session lingered.

### Root cause: race condition in `updateLRUTimestamp`

`updateLRUTimestamp` in `src/service-worker/cache-eviction.js` does a read-modify-write on a shared cache entry without serialisation:

```js
export async function updateLRUTimestamp(trackUrl) {
    const timestamps = await readTimestamps();
    timestamps[trackUrl] = Date.now();
    await writeTimestamps(timestamps);
}
```

It is called from `fetchTrack` (`src/service-worker/preload.js`) after every successful `trackCache.put()`. Preloads run in parallel via `tracks.forEach(async ...)`, so multiple concurrent invocations can interleave their reads and writes, losing updates. The sibling function `evictIfOverBudget` is already protected by a mutex (`evictionLock`) — for exactly the same reason — but the equivalent protection on `updateLRUTimestamp` was not added.

Effect: under sustained parallel-preload pressure (the normal mode of operation once the cache is full), the LRU timestamp store gradually loses fidelity. Eviction then starts removing tracks whose timestamps were lost rather than tracks that are genuinely oldest. Once the eviction ordering diverges enough, the track currently being preloaded for playback can itself be the eviction target, producing the observed thrash.

Filed as [`lucas42/lucos_media_seinn#456`](https://github.com/lucas42/lucos_media_seinn/issues/456) with a suggested fix (wrap `updateLRUTimestamp` in a serialising mutex, same pattern as `evictionLock`).

### Detection gap: degradation is silent

seinn currently has no in-page indicator when the SW cache is thrashing. The user sees their queue cycle through tracks rapidly with no audio. There is no warning banner, no automatic redirect of playback to a working device, no prompt to reload the tab. On 2026-05-19 and 2026-05-20 the only signals to the operator were:

1. The audible absence of music (user-observable, not monitored).
2. loganne `webhook-error-rate` alerts firing — which is a noisy *side effect*, not the actual symptom, and only fires if enough events accumulate failed webhooks.

The fix is defence-in-depth: detect repeated eviction storms inside the SW and surface a "playback degraded — reload required" UI indicator. Filed as [`lucas42/lucos_media_seinn#457`](https://github.com/lucas42/lucos_media_seinn/issues/457). This stands independently of the race fix — even after `#456` ships, Chrome's per-origin storage quota can drop unpredictably and other degradation modes will exist.

### Cascading effect: loganne webhook-error-rate alert

Each erroring track triggers a `trackUpdated` event from `lucos_media_metadata_api` → loganne, which fans out to webhook subscribers (`arachne.l42.eu/webhook`, `media-weighting.l42.eu/weight-track`, `ceol.l42.eu/webhooks/trackUpdated`). Under the burst (~15 erroring tracks/min × 3 webhook subscribers each = ~45 outbound fetches/min, with multi-second spikes far above that), loganne's HTTP client occasionally failed to establish outbound connections — error message `fetch failed`, which in undici is the generic class for `ECONNREFUSED` / `ENOTFOUND` / `ETIMEDOUT` / connection-pool-saturation conditions.

Important clarification on the router angle: the lucos_router nginx access logs show **every** loganne→webhook POST that *reached nginx* returning 2xx (210 × 204, 209 × 200, 208 × 202 in the 90-minute window). The failures happened **before** the request reached nginx — i.e. on loganne's outbound HTTP client side. So nginx was not the bottleneck. The likely loganne-side cause is undici connection-pool / DNS pressure during bursty fan-out, but pinning that down would need instrumentation that doesn't currently exist.

loganne's standing behaviour is to auto-retry a failed webhook once, then give up; restarting loganne does NOT re-attempt them. Clearing the resulting `webhook-error-rate` alert required a manual `POST /events/retry-webhooks` (33/33 events succeeded on first retry today — the downstreams had recovered).

### Broader pattern (not just these two days)

The same `trackUpdated ... errored` burst pattern appears in loganne's event history on:

| Date | Approx. window | Errored-track count in burst |
|---|---|---|
| 2026-05-13 | 14:00–14:20 UTC | 316 |
| 2026-05-15 | 11:50 + 22:10 | 91 |
| 2026-05-17 | 14:00 | 20 |
| 2026-05-18 | 14:40–14:50 | 79 |
| 2026-05-19 | 17:10–17:20 | 53 |
| 2026-05-20 | 10:50–11:10 | 196 |

This is a recurring multi-day pattern, consistent with the seinn cache-thrash mechanism progressively biting once any seinn tab is left running long enough to fill the cache. It is reasonable to assume earlier occurrences shared root cause even though we don't have the browser console logs to prove it.

---

## What Was Tried That Didn't Work

- Initial hypothesis: that the brief `502 Bad Gateway` from `ceol.l42.eu/v3/*` (seen during initial probing at ~11:12 UTC on 2026-05-20) was the cause of music not playing. Followed up by re-probing once the in-flight media_manager deploy finished — `/v3/*` came back to its normal 401 (auth-required) response. Media-manager was healthy throughout; the 502 was a transient mid-restart window, not a contributor to the music outage. The actual outage had been ongoing for at least 15 minutes before that 502 was seen.
- Considered framing the 11:12:25 deploy of `lucos_media_manager v1.0.31` ([PR `lucas42/lucos_media_manager#263`](https://github.com/lucas42/lucos_media_manager/pull/263) — parse JSON error payloads on `?action=error`) as part of the fix. It wasn't — the deviceSwitch at 11:12:14 (eleven seconds earlier) is what restored playback. The deploy is in-flight improved error visibility, not a remediation for this incident.
- Investigated `lucos_router` (nginx on avalon) as the cause of the loganne webhook `fetch failed` errors. nginx access logs showed every loganne POST that reached it returning 2xx; the error log had nothing relevant in the window. The failures originated on loganne's outbound HTTP client side, before any TCP connection reached the router. (Useful negative result — recorded here so the next investigation doesn't re-tread the same path.)

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix race condition in `updateLRUTimestamp` (root cause) | [`lucas42/lucos_media_seinn#456`](https://github.com/lucas42/lucos_media_seinn/issues/456) | Open |
| Detect and recover from cache-thrash state in the UI (defence-in-depth) | [`lucas42/lucos_media_seinn#457`](https://github.com/lucas42/lucos_media_seinn/issues/457) | Open |
| Enrich playback-error context sent from seinn to media-manager (already in flight, would have shortened diagnosis time) | [`lucas42/lucos_media_seinn#453`](https://github.com/lucas42/lucos_media_seinn/issues/453) | Open (pre-existing) |
| Send structured JSON error envelope from seinn to media-manager (receiver side already shipped) | [`lucas42/lucos_media_seinn#454`](https://github.com/lucas42/lucos_media_seinn/issues/454) | Open (pre-existing) |

The two pre-existing tickets (`#453`, `#454`) were filed before this incident; their existence is noted here because, had they been shipped, the diagnostic loop would have been much faster — seinn would have told media-manager *why* each track was erroring (cache miss / cache.put failure) rather than just *that* it errored.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
