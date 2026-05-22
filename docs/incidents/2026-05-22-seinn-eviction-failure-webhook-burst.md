# Incident: seinn cache-eviction TypeError silently breaks music playback; loganne event-loop saturation cascades upstream and downstream

| Field | Value |
|---|---|
| **Date** | 2026-05-22 |
| **Duration** | ~6h 16m user-visible playback failures (07:59:18 UTC → 14:14:57 UTC); loganne webhook-error-rate alert window 14:10:41 UTC → 14:34:12 UTC |
| **Severity** | Partial degradation (music playback only, internal-facing) with cascading webhook-error-rate alerts on loganne, and ~33% silent event drop from `lucos_media_metadata_api` to loganne during the burst |
| **Services affected** | `lucos_media_seinn` (web player) primary; `lucos_loganne` (HTTP server saturated, dropped inbound events and slowed outbound webhook deliveries); `lucos_media_metadata_api` (silently dropped events when loganne couldn't respond in 5 s) |
| **Detected by** | User report (lucas42 noticed by ear that music had stopped). loganne `webhook-error-rate` `monitoringAlert` also fired at 14:10:41 UTC, but as a lagging downstream symptom — saturation of loganne's HTTP server itself produced no direct alert. |

---

## Summary

A long-running `lucos_media_seinn` Chrome tab degraded into a state where service-worker cache eviction operations themselves threw `TypeError: Failed to fetch`, preventing further track preloads. Every subsequent track failed to decode, was reported as errored to `lucos_media_metadata_api`, and fanned out via `lucos_loganne` to three webhook subscribers — saturating loganne's single-threaded Node.js event loop. Saturation cascaded in three directions at once:

- **Downstream** — webhook deliveries to `media-weighting.l42.eu`, `ceol.l42.eu`, and `arachne.l42.eu` hit `fetch failed (ETIMEDOUT)`; 19 events ended up with permanently-failed webhooks after the once-auto-retry was exhausted, driving four `webhook-error-rate` `monitoringAlert`s.
- **Upstream** — `lucos_media_metadata_api`'s 5-second client timeout fired on 25 of 76 `POST /events` during the burst window. The Go client is fire-and-forget — no retry, no queue — so those 25 events were silently dropped and never reached webhook fan-out at all.
- **Self-health** — `GET /_info` polls from `lucos_monitoring` returned 499 (Client Closed Request) on 6 of 7 attempts during the burst, briefly tripping the `fetch-info` check and producing the "2 failing checks" `monitoringAlert` at 14:13:48 UTC. That `/_info` failure rate was the most direct signal loganne was overloaded, but it surfaced only as a single alert because of `failThreshold` throttling.

Recovery was a manual Chrome-tab restart by lucas42 at ~14:17 UTC, followed by an SRE-initiated `POST /events/retry-webhooks` at ~14:33 UTC to clear the 19 stranded events. `monitoringRecovery` fired at 14:34:12 UTC.

This incident has two **chronic** contributors (long-standing bugs that the burst surfaced but did not create) and two **acute** contributors (specific failure modes that fired during the burst). Both classes are documented below.

Follow-up investigation since the initial report (2026-05-22 14:25 UTC) has substantially expanded the picture — in particular, the original report attributed root cause to a cache-eviction `TypeError` in seinn's service worker, but direct SW-console probing falsified that hypothesis. The actual deeper root cause is a separate, chronic LRU-desync race in the same code area. The report has been amended to reflect the current understanding.

Source issues: see Follow-up Actions, below — eight items across four repositories.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| (chronic, pre-burst) | seinn's service-worker LRU timestamp store has drifted progressively out of sync with the track cache. By the burst, only ~14 of ~815 cached tracks have LRU records — the eviction subsystem can see less than 2% of the cache. Cache is running at ~8 GB, 10× over its declared 750 MB budget. Mechanism documented under "Analysis → Chronic root cause #1". |
| 07:59:18 | First `trackUpdated ... errored` event of the day ("A Good Day"). Beginning of the slow-burn degradation window. |
| 07:59 → 14:08 | Trickle of `trackUpdated ... errored` events. 70 successfully reach loganne; some additional events likely dropped silently between media-api and loganne (visible only as media-api's WARN log lines, not as recorded events). |
| 14:08:10 | First event with a permanently-failed webhook delivery (auto-retry exhausted). |
| 14:08:21 → 14:14:15 | media-api's `Error occured whilst posting to Loganne ... context deadline exceeded` repeats every 5–7 seconds — 25 occurrences total. Each corresponds to a 499 in avalon's nginx access log: loganne couldn't respond to `POST /events` within media-api's 5-second timeout. Of 76 `POST /events` attempts during this window, **51 succeeded (202) and 25 returned 499** — a 33% silent-drop rate at the producer→bus edge. |
| 14:09:03 | First `GET /_info` 499 — loganne's HTTP server failing to respond to lucos_monitoring's poll. 6 of 7 polls during the burst would return 499 (86% failure rate). |
| 14:10:41 | loganne `monitoringAlert`: `webhook-error-rate` failing. |
| 14:13:48 | loganne `monitoringAlert`: `fetch-info` *and* `webhook-error-rate` failing. The `fetch-info` flap is the `/_info` 499s breaching `failThreshold`. |
| 14:14:57 | Last `trackUpdated ... errored` event ("Wonderful Remark") — burst ends. |
| 14:15:08 | loganne `monitoringAlert` re-fires for `webhook-error-rate`. |
| 14:17:01 | lucas42 saves the seinn Chrome devtools console log and restarts the seinn Chrome tab. |
| 14:19:58 | First successful `trackUpdated ... finished playing` after the tab restart ("Mad World"). Playback restored. |
| 14:25:15 | loganne `monitoringAlert` re-fires once more — 19 stranded events still in `status: failure`. |
| ~14:33 | `lucos-site-reliability` triggers `POST /events/retry-webhooks` after sampling the failed-webhook distribution. 19/19 events succeed on first retry. |
| 14:34:12 | loganne `monitoringRecovery` — all checks healthy. |
| ~15:01 | (post-incident investigation) First SW-console probe of seinn's cache state: `tracks-v1: 815, LRU: 14, broken entries: 0 of 815`. The "broken cache entries from interrupted `cache.put`" hypothesis is falsified. The LRU desync surfaces as the actual chronic root cause. |
| ~15:25 | Second SW-console probe (after ~24 min of normal music playback): `tracks-v1: 823, LRU: 2`. `tracks-v1` grew rather than shrinking, while LRU shrank — direct production observation of the `_evictIfOverBudget` ↔ `updateLRUTimestamp` race scrubbing updates in real time. |

---

## Analysis

The incident is the result of **four contributing factors interacting**: two chronic (latent until the burst) and two acute (specific to the burst). Each is sufficient on its own to make the picture worse; the combination is what produced the user-visible outage.

### Chronic root cause #1 — LRU desync race in seinn's service worker

`src/service-worker/cache-eviction.js` has two functions that perform read-modify-write on a shared timestamp store: `updateLRUTimestamp()` (called per preload) and `_evictIfOverBudget()` (called per cache write, prunes orphans + evicts LRU candidates). After [`lucas42/lucos_media_seinn#459`](https://github.com/lucas42/lucos_media_seinn/pull/459) shipped on 2026-05-20, `updateLRUTimestamp` calls are serialised against each other via `timestampLock`. But `_evictIfOverBudget` holds a *different* lock (`evictionLock`), and its writes to the timestamp store don't coordinate with `timestampLock`-held writes. The race:

1. `_evictIfOverBudget` reads timestamps as `{A:1, B:2, ..., N:14}` into a local snapshot.
2. Concurrently, `updateLRUTimestamp(X)` runs (allowed — different lock), reads the same `{A:1, ..., N:14}`, writes back `{A:1, ..., N:14, X:now}`.
3. `evictTrack` (or the prune-orphans pass) inside `_evictIfOverBudget` writes back its local snapshot `{A:1, ..., N:14}` minus the evicted URLs — **X is gone from the persistent store**.

Every eviction pass that overlaps with a concurrent `updateLRUTimestamp` call quietly scrubs that update. Over time the LRU map shrinks toward the tracks that happen never to coincide with an eviction pass. By the burst, the map held ~14 entries for ~815 cached tracks; the eviction subsystem could only see ~2% of the cache, leaving it 10× over its declared 750 MB budget at ~8 GB.

This was confirmed directly in production via SW-console probing at ~15:01 and ~15:25 UTC (see Timeline). Filed as [`lucas42/lucos_media_seinn#472`](https://github.com/lucas42/lucos_media_seinn/issues/472) with a unified-lock fix proposal. This finding *supersedes* the original incident's framing of `#469` as root cause — `#469`'s `TypeError` is now understood as an acute layer on top of this chronic substrate.

### Chronic root cause #2 — media-api's fire-and-forget loganne client

`lucos_media_metadata_api/api/loganne.go` posts events via:

```go
var loganneHTTPClient = &http.Client{Timeout: 5 * time.Second}

func (loganne Loganne) post(...) {
    // ... marshal request ...
    if _, err := loganneHTTPClient.Do(req); err != nil {
        slog.Warn("Error occured whilst posting to Loganne", slog.Any("error", err))
    }
}
```

5-second timeout, no retry, no queue, no error returned to the caller. When loganne's event loop is saturated (as it was today), media-api hits the timeout, logs a warning, and **silently drops the event**. There is no further mechanism to recover it.

This contrasts with `lucos_loganne_pythonclient`, which has retry semantics. So an estate-wide expectation that "loganne clients should be resilient to transient loganne pressure" exists in the Python ecosystem but not the Go one. Architectural shape of the fix (inline retry in media-metadata-api vs. extracted `lucos_loganne_goclient`) is under consultation with `lucos-architect`; ticket to be filed once shape is decided.

The 25 events dropped today are the visible signal. Past bursts (see daily recurrence table) presumably dropped events similarly — they just weren't investigated.

### Acute contributor #1 — cache-eviction TypeError in seinn's service worker

During the burst, `_evictIfOverBudget()` itself threw `TypeError: Failed to fetch` 18 times (vs. 2 successful evictions). With eviction blocked AND the LRU map only tracking ~14 of ~815 entries (chronic #1), cache had no path to drain and new preloads could not complete. Every subsequent track failed to decode.

The original incident report identified this as root cause. SW-console probing after the burst (block 3: `broken entries: 0 of 815`) refuted the proposed mechanism — no persistent corruption in cache entries. The TypeError was therefore likely transient — storage-layer pressure during peak load — and may simply disappear once chronic #1 is fixed (which would drop the cache from 8 GB / 815 entries back toward 750 MB / ~75 entries, an order of magnitude less work per eviction pass). Tracked as [`lucas42/lucos_media_seinn#469`](https://github.com/lucas42/lucos_media_seinn/issues/469); proposed sequencing is to ship the #472 fix first, then observe whether the TypeError still recurs.

### Acute contributor #2 — loganne's single-threaded event loop saturating

Once trackErrored events started fanning out at 5–10 events/min × 3 webhook subscribers, loganne's Node.js event loop became the contended shared resource:

- **Egress** — outbound webhook fetches were held open by slow downstream consumers (media-weighting in particular has a synchronous handler doing up to 3 sequential HTTP round trips per delivery, with p50 baseline 3.3s vs peers' 200–470ms). Each in-flight fetch consumed event-loop attention.
- **Ingress** — new `POST /events` from media-api couldn't be processed in time. 25 of 76 attempts returned 499 (33% silent drop).
- **Self-health** — `GET /_info` returned 499 on 6 of 7 polls (86% failure rate). The one alert that captured this (`fetch-info` failing at 14:13:48) was throttled by `failThreshold` and arrived too late to be useful in real time.

The first two are direct consequences of event-loop saturation; the third is the most direct symptom loganne emitted — and the one we discovered only by comparing nginx access logs *after* the incident wrap-up was already underway. **Loganne has no self-reported saturation metrics today** — no event-loop lag, no in-flight delivery count, no response-time percentile. The only way to detect the burst in progress was the lagging `webhook-error-rate` outcome alert, which fires ~30s after delivery failures have already accumulated.

Filed as [`lucas42/lucos_loganne#484`](https://github.com/lucas42/lucos_loganne/issues/484) — proposed metrics (event-loop lag via `perf_hooks.monitorEventLoopDelay()`, in-flight outbound webhook count, recent `POST /events` response-time percentiles) and corresponding monitoring-side checks.

### Cascading effect — webhook delivery failures and the per-subscriber latency profile

Of the 51 events that reached loganne, fan-out continued to struggle. Per-subscriber webhook delivery latency for the burst:

| Subscriber | n | p50 | p99 | max | deliveries > 5 s | post-burst |
|---|---|---|---|---|---|---|
| `monitoring.l42.eu/suppress/clear` | 33 | 206 ms | 563 ms | 563 ms | 0 | — |
| `arachne.l42.eu/webhook` | 177 | 470 ms | 12 118 ms | 13 722 ms | 23 | recovers ≤ burst end |
| `ceol.l42.eu/webhooks/trackUpdated` | 177 | 1 398 ms | 11 788 ms | 12 258 ms | 24 | recovers ≤ burst end |
| **`media-weighting.l42.eu/weight-track`** | **177** | **3 256 ms** | **22 714 ms** | **23 963 ms** | **68** | **stays elevated ~40 min past burst end** |

media-weighting is uniquely outlying on both baseline (p50 3.3s) and post-burst persistence (>5 s deliveries continuing at 14:32, 14:37, 14:41, 14:44, 14:51 UTC). The cause: its `/weight-track` handler does up to three sequential HTTP round trips to media-api inline on the request path, with no internal queue, on a 4-thread waitress server. Under burst input the WSGI accept queue grows; under steady state, every delivery sits on that work for ~3 s.

[ADR-0006](../adr/0006-webhook-consumer-accept-202-enqueue.md) (accepted 2026-05-20, in direct response to the 2026-05-19 incident) names this exact pattern as the consumer-side fix: accept-202-enqueue. [`lucas42/lucos_media_weightings#236`](https://github.com/lucas42/lucos_media_weightings/issues/236) is the retrofit ticket.

The per-subscriber failure rates (74% ceol / 47% weightings / 37% arachne) and the data total of 30 failures across 19 events (consistent with the three subscribers failing independently at their own rates, not with one failure causing the others) jointly argue that the three subscribers are *each* more likely to fail during the burst at *their own* rate. The shared "more likely during burst" is loganne's event-loop saturation (acute #2), not a coupling between the three failures. Once `lucos_loganne#483` ships and `errorPhase` is preserved through retries, the next burst will let us distinguish connect-phase vs response-phase ETIMEDOUTs at fix-time, refining this further.

### Detection gap — no in-page signal in seinn

`lucos_media_seinn`'s cache-thrash banner (shipped via `#460` for the 2026-05-19 incident) detects sustained eviction storms by counting successful evictions in a sliding window. Today's failure mode was the inverse — evictions *failing* before completing — so the detector's counter never incremented and the banner never appeared. lucas42 had no in-page indication anything was wrong; the only signal was the audible absence of music. Tracked as [`lucas42/lucos_media_seinn#470`](https://github.com/lucas42/lucos_media_seinn/issues/470).

### Bonus finding — empty-time race producing 400s on ceol

`src/client/track-status-update.js` doesn't guard against `getTimeElapsed()` returning `undefined` (which happens whenever `currentAudio` exists but its `startTime` field hasn't been assigned — including after a failed `getBuffer` where the catch block doesn't clear `currentAudio`). The result is a periodic `PUT /v3/playlist/null/<uuid>/current-time` with an empty body, which ceol rejects with `400 Bad Request: Invalid time given "". Must be a number`. 18 occurrences in today's saved console log. Doesn't itself feed the loganne cascade (direct PUTs, no webhook chain), but is a real bug. Tracked as [`lucas42/lucos_media_seinn#471`](https://github.com/lucas42/lucos_media_seinn/issues/471).

### Daily recurrence pattern

loganne `webhook-error-rate` alerts continue to fire daily since the 2026-05-19/20 incidents. The data set has grown to:

| Date | Alert times (UTC) | Errored-track count in burst |
|---|---|---|
| 2026-05-19 | 17:09, 17:16, 17:22, 19:39, 19:48, 23:59 | 56 |
| 2026-05-20 | 00:15, 11:03, 11:08, 11:13 | 205 |
| 2026-05-21 | 13:08, 14:46, 15:01, 15:09, 22:24 | 16 |
| 2026-05-22 | 14:10, 14:13, 14:15, 14:25 | 70 (+ likely ~25 silently dropped at media-api → loganne) |

The `#459` race-fix shipped on 2026-05-20; the daily pattern persists after it, consistent with the chronic #1 LRU desync (which `#459` did not address) being the dominant recurrence driver.

---

## What Was Tried That Didn't Work

- **Initial framing: "this is a recurrence of `#456`."** Reasonable starting hypothesis given identical surface symptoms after the race-fix shipped. Falsified by counting successful-vs-failed eviction lines in the saved console log (2 vs 18, the inverse of 2026-05-19's 158-vs-0). Useful to record because anyone seeing the next "music stopped" report should not assume it's a 2026-05-19 recurrence without first checking that distribution.
- **#469 hypothesis: `response.arrayBuffer()` on a corrupt cache entry from an interrupted `cache.put`.** Falsified by direct SW-console probe (block 3 in `/tmp/seinn_sw_diagnostic.js`): `broken entries: 0 of 815`. Every cached track response reads cleanly. The TypeError site is still unconfirmed but is not persistent corruption.
- **Cascade hypothesis (3): "media-api is the shared bottleneck for the webhook subscribers' downstream calls."** Falsified by media-api's logs and loganne's nginx access log for the burst window. media-api was *processing* incoming PUTs throughout (the `Set Track Weighting` log lines continue at normal cadence), but was *itself a victim* of loganne saturation — 25 of its 76 outbound POSTs to loganne timed out. The shared bottleneck is loganne, not media-api. The cascade mental model was inverted in the initial report and has been corrected.
- **Framing the recovery as "the in-flight media_manager deploy".** Falsified by the timeline: the last `trackUpdated ... errored` is at 14:14:57 and the first `trackUpdated ... finished playing` is at 14:19:58, both bracketing the 14:17 Chrome-tab restart. The deploy at 14:25 — if any — is unrelated. Same conclusion as the prior incident.
- **Retrying stranded webhooks before sampling.** Operationally correct (cleared the alert) but cost us the per-attempt `errorPhase` field (just shipped in `lucos_loganne#480`) for these 19 events. Memory rule saved (`feedback_snapshot_before_retry.md`); `lucas42/lucos_loganne#483` (preserve attempt history) would make this rule obsolete by always keeping the data through retries.

---

## Follow-up Actions

Full inventory of issues raised by this incident, across all four repositories touched. Six are filed and tracked; one is in architectural consultation; one is filed elsewhere (this incident report itself, amended via PR linked at top of the issue list).

| # | Action | Issue / PR | Status |
|---|---|---|---|
| 1 | **Fix LRU desync race in service-worker cache eviction** — unified-lock fix proposed. The deeper root cause behind today's surface symptom, directly observed scrubbing LRU updates in production at ~15:25 UTC. | [`lucas42/lucos_media_seinn#472`](https://github.com/lucas42/lucos_media_seinn/issues/472) | Open |
| 2 | **Investigate cache-eviction TypeError** (original framing). Recommend re-observing after `#472` ships — the TypeError may not recur once the cache is back within budget. | [`lucas42/lucos_media_seinn#469`](https://github.com/lucas42/lucos_media_seinn/issues/469) | Open (revised) |
| 3 | **Extend cache-thrash banner to fire on failed-eviction storms**, not only successful-eviction storms. Closes the in-page detection gap from this incident. | [`lucas42/lucos_media_seinn#470`](https://github.com/lucas42/lucos_media_seinn/issues/470) | Open |
| 4 | **Fix empty-time race in `track-status-update.js`** producing 400s on ceol. Independent of the cascade story but surfaced during this investigation. | [`lucas42/lucos_media_seinn#471`](https://github.com/lucas42/lucos_media_seinn/issues/471) | Open |
| 5 | **Retrofit `lucos_media_weightings/weight-track` to accept-202-enqueue per ADR-0006.** Reduces loganne's outbound webhook hold time, indirectly addressing event-loop saturation on the ingress and `/_info` sides too. | [`lucas42/lucos_media_weightings#236`](https://github.com/lucas42/lucos_media_weightings/issues/236) | Open |
| 6 | **Surface loganne's own saturation state via `/_info` metrics** (event-loop lag, in-flight deliveries, response-time percentiles). Closes the "we had to compare logs at incident wrap-up to find this" gap. | [`lucas42/lucos_loganne#484`](https://github.com/lucas42/lucos_loganne/issues/484) | Open |
| 7 | **Preserve per-attempt webhook delivery history** instead of overwriting on retry. Closes the diagnostic-data-loss footgun (#483 body details the schema; SRE memory `feedback_snapshot_before_retry.md` becomes obsolete once shipped). | [`lucas42/lucos_loganne#483`](https://github.com/lucas42/lucos_loganne/issues/483) | Open |
| 8 | **Fix media-api's fire-and-forget loganne client** — 25 events silently dropped today. Architectural shape (inline retry in `api/loganne.go` vs extracted `lucos_loganne_goclient`) under consultation with `lucos-architect`; ticket to be filed once shape is decided. | (architect consultation, message 2026-05-22) | Pending design |

The fix proposed in [`lucas42/lucos_loganne#474`](https://github.com/lucas42/lucos_loganne/issues/474) (include `error.cause.code` in the persisted webhook errorMessage), filed in the 2026-05-19 report, has shipped. So have [`#479`](https://github.com/lucas42/lucos_loganne/issues/479) (`durationMs` per delivery) and [`#480`](https://github.com/lucas42/lucos_loganne/issues/480) (`errorPhase` distinguishing connect vs response ETIMEDOUT). All three pieces of diagnostic enrichment are in production; the only reason we couldn't use `errorPhase` for today's burst is the retry-overwrite problem `#483` will fix.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

---

## Amendment history

- **2026-05-22 (initial)**: Original report, framing root cause as the in-burst `TypeError` in `_evictIfOverBudget()`.
- **2026-05-22 (amended)**: Post-merge investigation revised the root-cause picture substantially. Direct SW-console probing falsified the "corrupt cache entry" hypothesis; the chronic LRU-desync race (`#472`) emerged as the deeper root cause. Avalon nginx access log analysis surfaced the upstream silent-drop story (media-api → loganne, 25/76 events lost during the burst) and the loganne self-saturation common-mode mechanism. Four further issues filed (`#472`, `#484`, plus architect consultation on media-api's loganne client). Cascade hypothesis (3) — "media-api is the bottleneck" — was falsified and removed from analysis.
