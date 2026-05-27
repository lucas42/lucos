# Incident: seinn playback-error thrash — decode/fetch failures loop, existing cache-thrash detector watched the wrong signal

| Field | Value |
|---|---|
| **Date** | 2026-05-26 |
| **Duration** | ~29 minutes (09:59 UTC → 10:28 UTC) |
| **Severity** | Partial degradation (music playback only, internal-facing) |
| **Services affected** | `lucos_media_seinn` (web player) primary; `lucos_loganne` (event-loop-lag-low check flapped 5x during the burst from outbound webhook fan-out, no inbound 499s this time) |
| **Detected by** | User report (lucas42 noticed by ear that music had stopped). `lucos_loganne` `event-loop-lag-low` `monitoringAlert` also fired five times during the window — at 10:01, 10:06, 10:18, 10:21, 10:25 UTC — but each was a brief flap that recovered before being actioned. |

Source issue: [`lucas42/lucos_media_seinn#482`](https://github.com/lucas42/lucos_media_seinn/issues/482) (closed by [PR #483](https://github.com/lucas42/lucos_media_seinn/pull/483), merged 14:19:31 UTC).

---

## Summary

On 2026-05-26 between 09:59 and 10:28 UTC, a long-running `lucos_media_seinn` Chrome tab entered a playback-error thrash loop: every track attempt either failed to fetch from the network or failed to decode, the manager auto-advanced to the next track, and the cycle repeated every ~2.4 s. 724 server-side `trackUpdated` "errored" events were recorded over 29 minutes. Recovery required a manual page reload by lucas42 at ~10:28; the [`lucas42/lucos_media_seinn#482`](https://github.com/lucas42/lucos_media_seinn/issues/482) fix (a new playback-error sliding-window detector that triggers the existing cache-thrash banner) shipped four hours later in [PR #483](https://github.com/lucas42/lucos_media_seinn/pull/483).

This is the **fourth seinn playback-failure incident** in seven days (2026-05-19, 2026-05-20, 2026-05-22, 2026-05-26) and a **mechanistically distinct failure mode** from the previous three: today's burst was client-side decode + fetch errors with no cache eviction involved, while the prior three were rooted in service-worker cache eviction problems. The existing cache-thrash banner (shipped via `lucas42/lucos_media_seinn#460` for the 2026-05-19 incident, extended via `lucas42/lucos_media_seinn#470` for the 2026-05-22 incident) watches CacheStorage eviction churn and could not see today's failure mode — hence no in-page signal until the user reloaded.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| (chronic, pre-burst) | Long-lived seinn Chrome tab open since at least the prior day. |
| 09:55–09:59 | Client `Failed to load resource: net::ERR_TIMED_OUT / NETWORK_IO_SUSPENDED / ADDRESS_UNREACHABLE` on `/v3/poll` — 79 occurrences captured in the saved console log (lines 1–235). Laptop's network connection was briefly disrupted. |
| 09:59:32 | First `trackUpdated ... errored` event of the burst (`lastErrorMessage: "Unable to decode audio data"`). |
| 10:01:00 | `lucos_loganne` `monitoringAlert`: `event-loop-lag-low` failing. Recovered 10:01:57. Knock-on load from ~25 trackUpdated events/min × 3 webhook subscribers each ≈ 75 outbound fires/min. |
| 10:01:10 | First `trackUpdated ... errored` with `lastErrorMessage: "Failed to fetch"`. From here onward both error types interleave; network had recovered but the rejected-promise cache in `src/client/buffers.js` (lines 27–31) keeps returning the same rejected fetch promise on subsequent attempts at the same URL. |
| 10:06, 10:18, 10:21, 10:25 | Four further `lucos_loganne` `event-loop-lag-low` flaps during the burst, each recovering within ~1 min. |
| 10:08:34 | lucas42 captures the Chrome dev-tools console log. (See `lucas42/lucos_media_seinn#482` body for breakdown.) |
| 10:27:44 | Last `Unable to decode audio data` event. |
| 10:28:24 | Last `Failed to fetch` event. lucas42 reloads the tab — burst ends. 724 total `trackUpdated` "errored" events recorded over the 29-minute window (336 decode + 388 fetch). |
| 10:45:29 | `lucos_media_seinn#482` filed. |
| 10:47:57 | Triaged: Status = Ready, Priority = High, Owner = lucos-developer. |
| 12:06:16 | `lucos-developer` posts implementation plan. |
| 14:19:31 | PR #483 merged — closes #482. |

---

## Analysis

### Acute root cause — client decode + fetch failures in a thrash loop

The mechanism is documented in detail in `lucas42/lucos_media_seinn#482`. Summarised:

```
playTrack(track)
  → await getBuffer(track.url)
      → fetch(url) … "Failed to fetch"          [Phase B — 388 events]
      → audioContext.decodeAudioData(bytes) … "Unable to decode audio data"   [Phase A — 336 events]
  → catch: console.error("Skipping track", …);  → DELETE …?action=error
  → media-manager.flagTrackAsError → playlist advances to next track
  → polling.js delivers new currentTrack to client
  → updateCurrentAudio → playTrack(next) → fails too → loop
```

Each cycle takes ~2.4 s — that's the observed thrash period (724 events / 29 min ≈ one error every 2.4 s).

The initial trigger was a transient ~4-minute network outage on lucas42's laptop (`net::ERR_TIMED_OUT / NETWORK_IO_SUSPENDED / ADDRESS_UNREACHABLE` between 09:55 and 09:59). But the burst continued for **another 25 minutes after the network recovered**, because of a secondary client bug:

`src/client/buffers.js` lines 27–31 caches rejected promises indefinitely. Once `getBuffer(url)` rejects, every subsequent call for the same URL returns the same rejected promise — no retry until the page reloads. The `preBufferTracks` path (lines 41–44) deletes the entry on failure, but the `playTrack → getBuffer` path does not. In practice the manager auto-advances to different URLs each cycle, so this rarely bites — but it is the reason a reload was needed and not just a "wait for network to come back" fix.

### Manager unconditionally auto-advances

`lucos_media_manager` advances the playlist on every `?action=error` (correct behaviour for a one-off bad track; pathological when *every* track this device tries fails). Server-side this is the right contract: only the client can distinguish "this device is broken" from "this track is broken". So the fix sits client-side.

### Detection gap — existing cache-thrash detector watches the wrong signal

`src/service-worker/cache-eviction.js` defines two sliding-window detectors:

1. `thrashDetector` — fires when ≥20 successful evictions occur in 60 s.
2. `failureDetector` — fires when ≥2 eviction failures occur in 60 s.

Both watch CacheStorage eviction churn. Today's failure mode produced **neither signal**: cache entries weren't being evicted — the bytes already in the cache were undecodable, *or* the network rejected the fetch before anything reached the cache. The service worker saw nothing wrong, and the `cache-thrash-banner` BroadcastChannel was never invoked.

The 2026-05-22 report's follow-up `#470` extended the detector to fire on **failed evictions** — orthogonal to today's failure mode. The right fix is a **third** detector watching playback errors directly, which is what `#482` / PR `#483` ships.

### Cascade — knock-on event-loop pressure on loganne

724 `trackUpdated` events × 3 webhook subscribers = ~2172 outbound webhook fires across the 29-minute window, peaking at ~75 fires/min. All 724 webhook deliveries themselves succeeded — **no `webhook-error-rate` alert this time**, in contrast to the 2026-05-22 incident where outbound deliveries timed out and ~25 inbound events were silently dropped.

But `lucos_loganne` did flap `event-loop-lag-low` 5 times during the burst. Each was brief (recovery within ~1 minute) and the check's `failThreshold: 2` correctly rode out single-poll transients. The `event-loop-lag-low` check itself was the right signal at the right specificity — shipped via [`lucas42/lucos_loganne#484`](https://github.com/lucas42/lucos_loganne/issues/484) (closed 2026-05-23) and tuned via [`lucas42/lucos_loganne#493`](https://github.com/lucas42/lucos_loganne/issues/493) (closed same day) — but no separate alert/escalation cascaded from this. The lesson is that the post-#484 instrumentation **worked**: we have a real-time signal for loganne saturation now, which was the gap the 2026-05-22 report had highlighted.

### Daily recurrence pattern (updated)

| Date | Mechanism | Errored-track count in burst | Recovery |
|---|---|---|---|
| 2026-05-19 | Cache thrash (LRU race surfacing once cache hit 750 MB budget) | 56 | `deviceSwitch` |
| 2026-05-20 | Cache thrash | 205 | `deviceSwitch` |
| 2026-05-21 | Cache thrash (minor occurrence — multiple shorter bursts at 13:08, 14:46, 15:01, 15:09, 22:24 UTC; no separate incident report filed, data included in the 2026-05-22 report's recurrence table) | 16 | — |
| 2026-05-22 | Cache-eviction `TypeError` ↑ chronic LRU desync; loganne cascade with 25 silent drops | 70 (+ ~25 dropped) | Manual tab reload + SRE-initiated webhook retry |
| **2026-05-26** | **Client decode + fetch errors; no cache eviction; no loganne outbound saturation** | **724** | **Manual tab reload** |

Four of the five bursts are now traced to a *cache* failure mode in the service worker. The 2026-05-26 burst is the outlier in that respect — and the reason the existing detectors had no chance.

---

## What Was Tried That Didn't Work

- **Initial framing: "another cache-thrash incident."** Reasonable given the surface symptoms (music stopped, many "errored" events) and the recency of the 2026-05-22 incident. Falsified by counting `lastErrorMessage` strings in the trackUpdated events: 336 decode + 388 fetch, **zero** cache-eviction signals in the captured client console log. Useful to record because the next "music stopped" report should not be reflex-classified into the cache-eviction bucket.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add playback-error thrash detector — fire cache-thrash banner on decode/fetch error loops. Closes the detection gap that let today's burst persist for 29 minutes without an in-page signal. | [`lucas42/lucos_media_seinn#482`](https://github.com/lucas42/lucos_media_seinn/issues/482) / [PR #483](https://github.com/lucas42/lucos_media_seinn/pull/483) | Done (merged 14:19:31 UTC) |

No further actions required. The rejected-promise pinning in `src/client/buffers.js` is described in the issue body but is in practice covered by reload-on-banner (which clears all SW state and rejected promises); raise separately only if it becomes an issue.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
