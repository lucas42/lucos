# Incident: Wake-up music didn't play because the media_manager queue stayed empty

| Field | Value |
|---|---|
| **Date** | 2026-10-08 |
| **Duration** | ~2h14m from the collection switch (07:54 UTC) to recovery (10:08 UTC). Monitoring was red for 2h12m of that (07:56 to 10:08 UTC) |
| **Severity** | Partial degradation |
| **Services affected** | lucos_media_manager (ceol.l42.eu), and every player it drives |
| **Tracking issue** | lucas42/lucos_media_manager#302 (open; second occurrence) |
| **Detected by** | Monitoring alerts at 07:56 UTC. Nobody acted on them. The SRE ops check found them on 2026-10-09 ~22:00 UTC, after the outage was over |

---

## Summary

At 07:54 UTC the bedroom "wake-up" scene in lucos_scenes switched media_manager to the `waking` collection and moved playback to the Bedroom device. Switching collection clears the queue and starts one background fetch of new tracks. media_manager was slow to handle the request, then the fetch ran for 21s and finished with **zero tracks queued**, as media_manager's own Loganne event records. Because media_manager never retries a failed fetch (lucas42/lucos_media_manager#302), the queue stayed empty and the Bedroom device had nothing to play for about 2h14m. The queue refilled at 10:08 UTC. We don't know what triggered the refill. This is the second occurrence of #302. The first, on 2026-09-29, lasted 31 minutes and ended when the container was restarted. The fix, retrying until the fetch succeeds, is already agreed on that ticket but hasn't shipped.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-09-29 07:18–07:49 | First occurrence of the same bug (31 min). Diagnosed on lucas42/lucos_media_manager#302, fix agreed, ticket left open |
| 2026-10-08 07:25 | Last deploy before the incident (`lucos_arachne`). Nothing deploys during the incident |
| 07:53:58 | lucos_scenes logs `action=playCollection collection=waking` and sends `PUT /v3/current-collection` to ceol |
| ~07:54:15 | media_manager reaches `switchFetcher` and the collection fetch starts, about 17s after scenes sent the request. Derived from the 07:54:36 event minus `firstBatchLatencyMs`; see Analysis |
| 07:54:17 | lucos_scenes logs `action=setVolume`. The collection PUT took about 19s to return |
| 07:54:18 | lucos_scenes logs `action=switchDevice device=:bedroom` |
| 07:54:34 | Loganne `deviceSwitch`: "Moving music to play on Bedroom" |
| 07:54:36 | Fetch finishes. Loganne `collectionSwitch` "Switched to collection Arise! ye workers from your slumbers", carrying **`collectionSize: 0`, `firstBatchLatencyMs: 21034`** |
| 07:56:18 | Monitoring alert: `lucos_media_manager` `empty-queue` |
| 07:56:20 | Monitoring alert: `empty-queue` and `queue` |
| 10:08:27 | Monitoring recovery: the queue has tracks again. Cause unknown; not a restart (the container's `StartedAt` is the next day) |
| 2026-10-08 15:19 | `lucos_router` restarts, clearing its access log for the incident window |
| 2026-10-09 07:14 | `lucos_media_manager` redeploys (lucas42/lucos_media_manager#304, Dependabot), clearing its log, including the GC and safepoint lines |
| 2026-10-09 ~22:00 | SRE ops check finds the alert in Loganne. The only container logs left covering the window are lucos_scenes and lucos_media_metadata_api, both started 2026-10-05 |

---

## Analysis

### Trigger: media_manager stalled, then the fetch queued nothing

**The fetch outcome is recorded directly** (lucos-architect). media_manager posts `collectionSwitch` from `switchFetcher`'s `afterFetch` callback, which runs once `fetcher.run()` returns, whether the fetch succeeded or failed. The 07:54:36.174 event carries `collectionSize: 0` and `firstBatchLatencyMs: 21034`. Each of the other 18 `collectionSwitch` events in Loganne from 2026-10-03 to 2026-10-09 has a collection size of 16–40 and a latency of 466–2367ms, including two switches to `waking` on 10-05 and 10-09. So the fetcher thread **finished** at 07:54:36 having queued nothing. It wasn't hung. That fits the `empty-queue` check, which stays green while a fetch is in progress and went red at 07:56:18 (lucos-developer).

**How the fetch ended can't be determined.** There are two possibilities: an exception, which media_manager logs as `ERROR: Can't fetch new tracks.` (the 09-29 failure looked like this), or a 200 with an empty `tracks` array, which logs nothing. media_manager's log went in the 10-09 redeploy. lucos_media_metadata_api's log does survive, but it doesn't record GET requests. The 21s latency matches neither of media_manager's timeouts (5s connect, 30s read, in `MediaApi.java`), so it gives no clue either.

**There were two slow stretches** (lucos-architect). The `/v3/current-collection` handler makes no network call before `switchFetcher` and returns straight after it. Subtracting 21.034s from the event time puts the start of `switchFetcher` at about 07:54:15. Scenes sent the PUT at 07:53:58 (its calls run in sequence with no sleep, per `playCollectionAtVolumeOnDevice` in lucos_scenes `src/lucos_scenes/media_controls.clj`) and got the response at about 07:54:17. So:

1. About **17s passed before media_manager reached the handler** (07:53:58 to 07:54:15). That is a stall inside media_manager, or on the way to it.
2. Then a **21s fetch that queued nothing** (07:54:15 to 07:54:36).

The `deviceSwitch` event lags scenes' `switchDevice` call by about 16s (07:54:18 to 07:54:34), which fits a stall that was still going on. This breakdown assumes the clocks on scenes and media_manager agree to within about a second, which hasn't been checked.

The collection isn't the problem. `waking` was switched to eight other times between 2026-09-17 and 2026-10-05, and again on 2026-10-09, with no alert.

The 09-29 occurrence also involved a stalled JVM, but there the stall was caused by the last stage of a 3.48GB image pull (see the comments on lucas42/lucos_media_manager#302). **That explanation doesn't fit here:** nothing deployed between 07:25 and 10:08. What stalled media_manager this time is **not known**. The GC and safepoint logging added in lucas42/lucos_media_manager#299 would have shown it, but that log was lost in the 10-09 redeploy. One possible contributor, not verified: lucos_media_metadata_api spent all of 10-08 from about 00:30 to after 12:00 handling a steady stream of about 1,200 fingerprint-keyed `PUT` track updates an hour (most likely a lucos_media_import scan). That might explain a slow or empty `/random` response. It doesn't explain the 17s before media_manager reached the handler, and no other fetch ran during the import to compare against.

### Main contributing factor: a failed fetch is never retried

This is lucas42/lucos_media_manager#302, confirmed from the code there. A `CollectionFetcher` thread whose fetch fails logs the error and exits. One that gets an empty response exits without logging anything. Every other trigger that refills the queue needs a request from outside, or a restart, and with nothing playing, usually nothing comes. One failed fetch therefore becomes an outage of indefinite length. On 09-29 it was 31 minutes, and only because someone restarted the container. This time it was 2h14m.

That makes the trigger matter much less, with one caveat. If the fetch threw an exception, retrying with the agreed 5s/10s/20s/40s/60s backoff would most likely have refilled the queue within a minute or two, not two hours. If it got an empty 200, the agreed design stops on success, so it would *not* have retried. That is why lucos-architect recommended on #302 that a fetch which queues zero tracks also counts as a failed attempt. With that change, the fix covers both possible endings.

### Detection: the alert fired and nobody acted on it

Monitoring alerted within two minutes and stayed red for the whole outage. Nobody responded until an ops check the next evening. That is the alert-to-action gap already settled on lucas42/lucos#290, so this report doesn't propose anything new for it.

What recovered the queue at 10:08:27 is unknown. The lucos_scenes log has no action between 07:54:18 and 10:15, and the router and media_manager logs that would show which client it was are gone. It wasn't a switch to a different collection, because Loganne has no `collectionSwitch` between 07:54:36 and 21:26. It wasn't a same-slug switch either, because that returns `204 Not Changed` without fetching. The set of possible triggers is wider than #302's body suggests (lucos-architect). `removeTrackByUuid()` (behind the complete, skip-by-uuid and error POSTs), `skipTrack()` and `deleteTrackById()` all call `topupTracks()` unconditionally. They do it even when the queue is empty or the uuid isn't in it. So any of those requests from a player, even for a stale track, or a track-deletion notification from the media API, would have refilled the queue. Which of these it was is **unknown**.

### Evidence loss

Two routine restarts cleared the logs for the incident window: the router at 2026-10-08 15:19 and media_manager at 2026-10-09 07:14. Both happened before anyone looked. The only logs left covering the window belonged to the two containers that happened not to restart (lucos_scenes and lucos_media_metadata_api), and neither held the deciding evidence. An incident that isn't picked up the same day will usually have lost its container logs. That problem is already tracked on lucas42/lucos#214 (open: a durable logging strategy), and I've added this incident to it as evidence.

---

## What Was Tried That Didn't Work

- **Host-level metrics for avalon at 07:53–07:55Z** (lucos-system-administrator). None exist. avalon has no sysstat or atop history, and the agent user can't read the journal (lucas42/lucos_agent_coding_sandbox#99). An OOM kill or IO stall can't be ruled out. `docker events` for 07:45–08:05Z returned nothing, which weakly supports "no deploy or pull". It isn't proof, because the depth of the event buffer wasn't verified.
- **Reading media_manager's and the router's logs for the window.** Both had been cleared by restarts. See Evidence loss above.
- **lucos_media_metadata_api's log**, to tell an exception from an empty 200. The log survives, but it doesn't record GET requests.
- **The 09-29 image-pull explanation.** I checked it against Loganne's deploy events and ruled it out for this occurrence.

---

## Follow-up Actions

Considered and not filed:

- **A host metrics recorder on avalon (sysstat)**, suggested by lucos-system-administrator. It would have answered the question of what stalled media_manager, here and on 09-29. But the answer wouldn't change the fix, which works whatever caused the stall. The trigger for revisiting this is a third unexplained stall on avalon that is *not* covered by an existing fix.
- **An explicit empty-queue or failed-fetch state in the player UI, and a failure signal from the wake-up scene**, suggested by lucos-ux. For a wake-up scene the person is asleep, so a UI state wouldn't have helped. Once #302 lands, an empty queue retries by itself instead of persisting. Revisit if the `empty-queue` alert stays red for more than 10 minutes again, whether or not #302 has shipped by then.

| Action | Issue / PR | Status |
|---|---|---|
| Retry failed track fetches until they succeed, with backoff, and report a "retrying" state to monitoring. lucos-architect has recommended that a fetch which queues zero tracks also counts as a failed attempt | lucas42/lucos_media_manager#302 | Open (fix agreed 2026-09-29, not yet implemented) |
| Put a timeout on media API response bodies. A body that stalls after its headers would hang the fetcher while `/_info` stays green (latent; not what happened here) | lucas42/lucos_media_manager#305 | Open |
| Durable container logs, so restarts don't erase incident evidence | lucas42/lucos#214 | Open (evidence added) |
| lucos_scenes sends no `User-Agent` to ceol (ADR-0001), found while investigating this incident | lucas42/lucos#251 (remediation batch) | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
