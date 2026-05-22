# Incident: seinn cache-eviction TypeError silently breaks music playback

| Field | Value |
|---|---|
| **Date** | 2026-05-22 |
| **Duration** | ~6h 16m (07:59:18 UTC to 14:14:57 UTC of user-visible playback failures); webhook-error-rate alert window 14:10:41 UTC to 14:34:12 UTC |
| **Severity** | Partial degradation (music playback only, internal-facing) with cascading webhook-error-rate alerts on loganne |
| **Services affected** | `lucos_media_seinn` (web player) primary; `lucos_loganne` (downstream webhook-error-rate) |
| **Detected by** | User report (lucas42 noticed by ear that music had stopped). The loganne `webhook-error-rate` `monitoringAlert` also fired at 14:10:41 UTC — that signal is a downstream cascade indicator, not the actual primary symptom (per the same observation in the 2026-05-19 incident report). |

---

## Summary

A long-running `lucos_media_seinn` Chrome tab degraded into a state where service-worker cache eviction operations themselves threw `TypeError: Failed to fetch` on their internal Cache API calls, preventing any further tracks from being preloaded. Every subsequent track failed to decode, was reported as errored to `lucos_media_metadata_api`, fanned out as `trackUpdated` events through `lucos_loganne` to three webhook subscribers, and produced four `webhook-error-rate` `monitoringAlert`s on loganne. Recovery was a manual Chrome-tab restart by lucas42 after he noticed playback had stopped, followed by an SRE-initiated `POST /events/retry-webhooks` to clear 19 stranded webhook events.

The user-visible failure mode is identical to the [2026-05-19 incident](2026-05-19-seinn-cache-thrash-music-outages.md), but the underlying mechanism is distinct: that one was cache *thrash* (many successful evictions in a tight loop after the LRU race scrambled ordering); this one is the *inverse*, with eviction attempts failing entirely. The race-condition fix shipped in [`lucas42/lucos_media_seinn#459`](https://github.com/lucas42/lucos_media_seinn/pull/459) (closing [`lucas42/lucos_media_seinn#456`](https://github.com/lucas42/lucos_media_seinn/issues/456)) on 2026-05-20 would not have prevented this. The thrash-detection banner from [`lucas42/lucos_media_seinn#460`](https://github.com/lucas42/lucos_media_seinn/pull/460) (closing [`lucas42/lucos_media_seinn#457`](https://github.com/lucas42/lucos_media_seinn/issues/457)) also did not fire, because its detector counts only successful evictions — which is the gap tracked separately in the follow-up.

Source issues for this report: [`lucas42/lucos_media_seinn#469`](https://github.com/lucas42/lucos_media_seinn/issues/469) (root-cause investigation) and [`lucas42/lucos_media_seinn#470`](https://github.com/lucas42/lucos_media_seinn/issues/470) (banner detection gap).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 07:59:18 | First `trackUpdated ... errored` event of the day from `lucos_media_metadata_api` ("A Good Day"). Beginning of the slow-burn degradation window. |
| 07:59 → 14:08 | Trickle of `trackUpdated ... errored` events. 70 total across the window, peaking in the final ~15 minutes. Each errored event fans out to three webhook subscribers (`arachne.l42.eu/webhook`, `media-weighting.l42.eu/weight-track`, `ceol.l42.eu/webhooks/trackUpdated`). |
| 14:08:10 | Earliest event with a permanently-failed webhook delivery (auto-retry-once exhausted). |
| 14:10:41 | loganne `monitoringAlert`: `webhook-error-rate` failing. |
| 14:12:33 | Latest event with a permanently-failed webhook delivery. |
| 14:13:48 | loganne `monitoringAlert`: `fetch-info` *and* `webhook-error-rate` failing. The `fetch-info` flap is consistent with loganne's HTTP client under pressure from outbound retries (same pattern as 2026-05-20 at 11:08). |
| 14:14:57 | Last `trackUpdated ... errored` event ("Wonderful Remark") — the end of the burst. |
| 14:15:08 | loganne `monitoringAlert` re-fires for `webhook-error-rate` (events still in `status: failure`). |
| 14:17:01 | lucas42 saves the Chrome devtools console log (`/tmp/seinn.l42.eu-1779459421090.log`) and manually restarts the seinn Chrome tab. |
| 14:19:58 | First successful `trackUpdated ... finished playing` after the tab restart ("Mad World"). Playback restored. |
| 14:25:15 | loganne `monitoringAlert` re-fires once more — the 19 stranded `status: failure` events keep `webhook-error-rate` above threshold even after the actual source of errors has stopped. (Standing loganne behaviour: auto-retry once, then permanent failure; no fresh retry on monitoring poll.) |
| ~14:33 | `lucos-site-reliability` samples the failed-webhook distribution: all 19 events × 30 webhook deliveries fail with `fetch failed (ETIMEDOUT)` (matching the 2026-05-20 distribution, where retry was successful), and triggers `POST /events/retry-webhooks`. 19/19 events succeed on first retry. |
| 14:34:12 | loganne `monitoringRecovery` — all checks healthy. |

---

## Analysis

### Triggering condition: cache eviction itself throwing `TypeError: Failed to fetch`

The saved console log shows the following pattern counts:

| Pattern | Count |
|---|---|
| `web-player.js:92 Skipping track Unable to decode audio data` | 59 |
| `web-player.js:92 Skipping track Failed to fetch` | 3 |
| `cache-eviction.js:321 Cache eviction failed: TypeError: Failed to fetch` | **18** |
| `cache-eviction.js:247 Cache eviction: removed <url>` | **2** |
| `manager.js:41 PUT /v3/playlist/null/<uuid>/current-time → 400` | 18 |

Compared with the 2026-05-19 log (158 successful evictions, 0 failed), today's distribution is inverted: nearly every eviction attempt threw. Successful evictions are the precondition for normal preload-and-play; failing evictions leave the cache stuck over budget, so `trackCache.put()` cannot store new tracks, every preload fails, every track fails to decode at play time, and the manager keeps marking tracks errored as it walks the queue.

Source code in scope: `src/service-worker/cache-eviction.js`, specifically `_evictIfOverBudget()` (called from `preload.js:55` after every successful `trackCache.put()`). The `_evictIfOverBudget()` body does not itself call `fetch()` — it reads from caches via `cache.match()` / `response.arrayBuffer()` / `response.json()` and writes via `cache.put()` / `cache.delete()`. The most likely throwing site is `response.arrayBuffer()` inside `getCacheSizeWithMap()` on a cached track entry whose original `cache.put(request, fetchResponse)` (in `preload.js:53`) was interrupted mid-stream by a network drop, leaving a structurally-broken entry that throws on read. Confirming this hypothesis is the work tracked in [`lucas42/lucos_media_seinn#469`](https://github.com/lucas42/lucos_media_seinn/issues/469).

### Independence from the 2026-05-19 race fix

The race condition fixed by [`lucas42/lucos_media_seinn#459`](https://github.com/lucas42/lucos_media_seinn/pull/459) was about lost updates to the LRU timestamp store causing eviction to pick the *wrong* track. Today's failure is upstream of that: eviction can't pick *any* track because the eviction pass throws before reaching the candidate-selection loop. The two mechanisms are independent, even though they share the user-visible symptom.

### Detection gap: the cache-thrash banner did not fire

The thrash detector in `src/service-worker/cache-eviction.js` calls `recordEviction()` only after a successful `evictTrack()`. The outer wrapper consumes any error from `_evictIfOverBudget()`:

```js
export function evictIfOverBudget() {
    evictionLock = evictionLock
        .then(() => _evictIfOverBudget())
        .catch(err => console.error('Cache eviction failed:', err));   // L321
    return evictionLock;
}
```

When `_evictIfOverBudget()` threw 18 times today, the sliding-window counter `recentEvictionTimes` stayed empty, `thrashDetected` stayed `false`, and the `cache-thrash` `BroadcastChannel` message was never posted. The page client therefore never displayed `<cache-thrash-banner>`. lucas42 had no in-page signal that anything was wrong — only the user-perceptible audible silence of music having stopped. This is the gap tracked by [`lucas42/lucos_media_seinn#470`](https://github.com/lucas42/lucos_media_seinn/issues/470).

### Cascading effect: loganne webhook-error-rate alert

Same cascade pattern documented in the 2026-05-19 report. Each errored track produces a `trackUpdated` event from `lucos_media_metadata_api` → loganne, fanning out to three webhook subscribers. Under the burst, loganne's outbound HTTP client returns `fetch failed (ETIMEDOUT)` — the `error.cause.code` enrichment from [`lucas42/lucos_loganne#474`](https://github.com/lucas42/lucos_loganne/issues/474) (filed during the 2026-05-19 incident) is now visible in the persisted `webhooks[].errorMessage`, giving us the specific underlying code rather than the bare `fetch failed`. ETIMEDOUT here means the TCP connection didn't establish in time, not that a downstream returned an error; the downstreams were healthy when probed later, and the 19 stranded events all succeeded on retry. The 2026-05-19 report's note that nginx is not the bottleneck (the failures are on loganne's outbound undici side) applies again.

loganne's standing single-retry-then-permanent-failure behaviour meant the `webhook-error-rate` alert persisted for ~24 minutes from first to last `monitoringAlert` even though the actual error source had stopped 12 minutes earlier with the Chrome tab restart. Clearing the alert required the manual `POST /events/retry-webhooks`.

### Daily recurrence pattern

Loganne `webhook-error-rate` alerts have been firing daily since the 2026-05-19/20 incidents:

| Date | Alert times (UTC) | Errored-track count in burst |
|---|---|---|
| 2026-05-19 | 17:09, 17:16, 17:22, 19:39, 19:48, 23:59 | 56 |
| 2026-05-20 | 00:15, 11:03, 11:08, 11:13 | 205 |
| 2026-05-21 | 13:08, 14:46, 15:01, 15:09, 22:24 | 16 |
| 2026-05-22 | 14:10, 14:13, 14:15, 14:25 | 70 |

The `#459` race-fix shipped on 2026-05-20; the daily pattern continues after it, consistent with this incident's finding that the new mechanism is independent of the race. Whatever is producing the `TypeError: Failed to fetch` inside `_evictIfOverBudget()` is now the dominant recurrence driver.

---

## What Was Tried That Didn't Work

- **Initial framing: "this is a recurrence of #456."** Reasonable starting hypothesis given identical surface symptoms and ongoing webhook-error-rate alerts after the race-fix shipped. Falsified by counting eviction lines in today's console log — only 2 successful evictions versus 18 failures, the inverse of the 2026-05-19 thrash pattern. The race-fix is still working; this is a different mechanism that happens to surface as the same downstream behaviour. Useful to record because anyone seeing the next "music stopped" report should not assume it's a 2026-05-19 recurrence without first checking the successful-vs-failed eviction count in the saved console log.
- **Considered framing the burst-end as a result of media-manager or ceol load recovery rather than the Chrome tab restart.** Falsified by the timeline: the last `trackUpdated ... errored` is at 14:14:57, and the first `trackUpdated ... finished playing` is at 14:19:58 — both bracketed by the 14:17 file save / restart. Recovery is plainly Chrome-tab-restart-attributable, identical to the 2026-05-20 occurrence resolved by `deviceSwitch`.
- No diagnostic attempt was made to inspect the broken cache entries directly (e.g. via `chrome://serviceworker-internals` or by inspecting `caches.match()` results in the live tab). That investigation would have required catching the tab before lucas42 restarted it; it's now closed off until the next occurrence. The diagnostic hooks needed to capture this state at thrash-detect time exist in [`lucas42/lucos_media_seinn#460`](https://github.com/lucas42/lucos_media_seinn/pull/460) but, as established above, the detector never fired for this failure mode.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Investigate root cause of `TypeError: Failed to fetch` inside `_evictIfOverBudget()`; harden eviction against corrupt cache entries or fix the upstream cause (interrupted `cache.put` during preload). | [`lucas42/lucos_media_seinn#469`](https://github.com/lucas42/lucos_media_seinn/issues/469) | Open |
| Extend the cache-thrash banner detector to also fire on sustained eviction *failures*, not only successful-eviction storms, so the user gets an in-page signal in both failure modes. | [`lucas42/lucos_media_seinn#470`](https://github.com/lucas42/lucos_media_seinn/issues/470) | Open |
| Fix the empty-time race in `track-status-update.js` that produces 18 spurious 400s on `ceol.l42.eu/v3/playlist/null/<uuid>/current-time` during this burst (and any other burst where many tracks fail to decode). Independent of the eviction story but surfaced during this investigation. | [`lucas42/lucos_media_seinn#471`](https://github.com/lucas42/lucos_media_seinn/issues/471) | Open |

The fix proposed in [`lucas42/lucos_loganne#474`](https://github.com/lucas42/lucos_loganne/issues/474) (include `error.cause.code` in the persisted webhook errorMessage) has shipped since the 2026-05-19 report — visible in this incident's data as the explicit `(ETIMEDOUT)` qualifier. Useful diagnostic upgrade; carried over here for the audit trail.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
