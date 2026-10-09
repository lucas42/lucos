# Incident: Wake-up music didn't play because the media_manager queue stayed empty

| Field | Value |
|---|---|
| **Date** | 2026-10-08 |
| **Duration** | ~2h14m (07:54 UTC to 10:08 UTC) |
| **Severity** | Partial degradation |
| **Services affected** | lucos_media_manager (ceol.l42.eu), and every player it drives |
| **Tracking issue** | lucas42/lucos_media_manager#302 (open; second occurrence) |
| **Detected by** | Monitoring alerts at 07:56 UTC. Nobody acted on them. The SRE ops check found them on 2026-10-09 ~22:00 UTC, after the outage was over |

---

## Summary

At 07:54 UTC the bedroom "wake-up" scene in lucos_scenes switched media_manager to the `waking` collection and moved playback to the Bedroom device. Switching collection clears the queue and starts one background fetch of new tracks. That fetch never filled the queue. Because media_manager never retries a failed fetch (lucas42/lucos_media_manager#302), the queue stayed empty and the Bedroom device had nothing to play for about 2h14m. The queue refilled at 10:08 UTC. We don't know what triggered the refill. This is the second occurrence of #302. The first, on 2026-09-29, lasted 31 minutes and ended when the container was restarted. The fix, retrying until the fetch succeeds, is already agreed on that ticket but hasn't shipped.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-09-29 07:18–07:49 | First occurrence of the same bug (31 min). Diagnosed on lucas42/lucos_media_manager#302, fix agreed, ticket left open |
| 2026-10-08 07:25 | Last deploy before the incident (`lucos_arachne`). Nothing deploys during the incident |
| 07:53:58 | lucos_scenes logs `action=playCollection collection=waking` and sends `PUT /v3/current-collection` to ceol |
| 07:54:17 | lucos_scenes logs `action=setVolume`. The collection PUT took about 19s to return (see Analysis) |
| 07:54:18 | lucos_scenes logs `action=switchDevice device=:bedroom` |
| 07:54:34 | Loganne `deviceSwitch`: "Moving music to play on Bedroom" |
| 07:54:36 | Loganne `collectionSwitch`: "Switched to collection Arise! ye workers from your slumbers", 38s after scenes sent the request |
| 07:56:18 | Monitoring alert: `lucos_media_manager` `empty-queue` |
| 07:56:20 | Monitoring alert: `empty-queue` and `queue` |
| 10:08:27 | Monitoring recovery: the queue has tracks again. Cause unknown; not a restart (the container's `StartedAt` is the next day) |
| 2026-10-08 15:19 | `lucos_router` restarts, clearing its access log for the incident window |
| 2026-10-09 07:14 | `lucos_media_manager` redeploys (lucas42/lucos_media_manager#304, Dependabot), clearing its log, including the GC and safepoint lines |
| 2026-10-09 ~22:00 | SRE ops check finds the alert in Loganne. lucos_scenes' log (started 2026-10-05) is the only container log left that covers the window |

---

## Analysis

> **Note on cross-repo references:** All incident reports live in the `lucos` repo. Always use the fully-qualified format `lucas42/repo_name#N`.

### Trigger: the collection fetch at 07:54 didn't fill the queue (cause unknown)

The direct evidence is that the queue was empty from the collection switch until 10:08. The `empty-queue` check has no `debug` text, and the error media_manager would have logged (`ERROR: Can't fetch new tracks.`) went with the container log when it redeployed on 2026-10-09. So the claim that **the fetch failed** is inferred from the result. I didn't observe the failure itself.

The collection isn't the problem. `waking` ("Arise! ye workers from your slumbers") was switched to eight other times between 2026-09-17 and 2026-10-05, and again on 2026-10-09, with no alert.

There is circumstantial evidence that **media_manager was stalled or very slow at 07:54**:

- `playCollectionAtVolumeOnDevice` (lucos_scenes `src/lucos_scenes/media_controls.clj`) makes its three calls one after another on a single thread, with no sleep between them. The 19s gap between the `playCollection` and `setVolume` log lines is therefore almost all the time `PUT /v3/current-collection` took to return.
- media_manager's own `collectionSwitch` event is stamped 07:54:36, 38s after scenes sent the request.

The 09-29 occurrence had the same shape, and there the JVM was shown to be stalled during the last stage of a 3.48GB image pull (see the comments on lucas42/lucos_media_manager#302). **That explanation doesn't fit here:** nothing deployed between 07:25 and 10:08. What stalled media_manager this time is **not known**. The GC and safepoint logging added in lucas42/lucos_media_manager#299 would have shown it, but that log was lost in the 10-09 redeploy.

### Main contributing factor: a failed fetch is never retried

This is lucas42/lucos_media_manager#302, confirmed from the code there. A failed `CollectionFetcher` thread logs the error and exits. Every other trigger that refills the queue (track complete, skip, startup) needs something to happen first, and with nothing playing, nothing does. One failed fetch therefore becomes an outage of indefinite length. On 09-29 it was 31 minutes, and only because someone restarted the container. This time it was 2h14m.

That makes the trigger matter much less. However long the stall lasted, retrying with the agreed 5s/10s/20s/40s/60s backoff would most likely have refilled the queue within a minute or two, not two hours.

### Detection: the alert fired and nobody acted on it

Monitoring alerted within two minutes and stayed red for the whole outage. Nobody responded until an ops check the next evening. That is the alert-to-action gap already settled on lucas42/lucos#290, so this report doesn't propose anything new for it.

What recovered the queue at 10:08:27 is unknown. The lucos_scenes log has no action between 07:54:18 and 10:15, and the router and media_manager logs that would show which client it was are gone. Going by the refill triggers listed on #302, it was probably a client action such as a skip or a play/pause from a player, but I haven't verified that.

### Evidence loss

Two routine restarts cleared the logs for the incident window: the router at 2026-10-08 15:19 and media_manager at 2026-10-09 07:14. Both happened before anyone looked. Only lucos_scenes, which hadn't restarted since 2026-10-05, kept a usable record. An incident that isn't picked up the same day will usually have lost its container logs. This report records that constraint but doesn't propose a fix for it.

---

## What Was Tried That Didn't Work

- **Reading media_manager's and the router's logs for the window.** Both had been cleared by restarts. See Evidence loss above.
- **The 09-29 image-pull explanation.** I checked it against Loganne's deploy events and ruled it out for this occurrence.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Retry failed track fetches until they succeed, with backoff, and report a "retrying" state to monitoring | lucas42/lucos_media_manager#302 | Open (fix agreed 2026-09-29, not yet implemented) |
| lucos_scenes sends no `User-Agent` to ceol (ADR-0001), found while investigating this incident | lucas42/lucos#251 (remediation batch) | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
