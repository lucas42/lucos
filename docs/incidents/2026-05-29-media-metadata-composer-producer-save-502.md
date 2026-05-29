# Incident: Track saves 502 when setting composer/producer (synchronous eolas dependency)

| Field | Value |
|---|---|
| **Date** | 2026-05-29 |
| **Duration** | First reported ~15:11 UTC; **still failing in production at time of writing** (persistent, not a transient window) |
| **Severity** | Partial degradation — user-blocking for any track edit that sets a composer/producer tag |
| **Services affected** | lucos_media_metadata_api (save path), lucos_media_metadata_manager (save UX + error rendering), lucos_eolas (degraded dependency) |
| **Detected by** | User report — lucas42, via team-lead (blocked saving track 22829) |

> **Correction note (supersedes the first version of this report).** The initial version of this report (merged in `lucas42/lucos#202`) framed this as a *transient* eolas-degradation window (~15:07–15:11 UTC) that had likely self-recovered. **That was wrong.** lucas42 retried and the save **still fails** — this is a **persistent, reproducible** failure on the composer/producer save path, not a transient blip. This version corrects the Summary, Analysis, Timeline, and Resolution accordingly.

---

## Summary

Saving a track that **sets a composer or producer tag** consistently fails with **HTTP 502 Bad Gateway**. Root cause: PR `lucas42/lucos_media_metadata_api#274` (the composer/producer → `eolas:Person` migration) added a **synchronous eolas call to the track-save request path** for these two predicates. When that eolas resolve is slow (eolas's bulk endpoint is currently ~23s — `lucas42/lucos_eolas#283`), the save hangs and a 502 is returned. lucas42 hit this saving track 22829; it is **still failing in production** as of this writing. The durable fix (make the save-path eolas dependency resilient) is tracked in `lucas42/lucos_media_metadata_api#278`; the eolas-side slowness in `lucas42/lucos_eolas#283`. Immediate workaround: save the track **without** touching composer/producer.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-29 11:35 | `lucas42/lucos_media_metadata_api#274` (composer/producer → eolas:Person) merged; deployed as 1.0.62 |
| ~15:11 | lucas42 reports saving track 22829 fails ("API returned unexpected status code 400" — later confirmed a manager mislabel; real status 502). API logs `update/create track 22829` then no completion |
| 15:33 | Retry — same `PATCH /tracks/22829` → **502**, API hangs again (logs `update/create track` then silence) |
| ~15:21–15:45 | SRE diagnosis: eolas fast paths healthy (create 43ms / 0.16s, person-list 0.22s); **bulk `/metadata/all/data/` ~23s**; 22829 has **no stored composer/producer/memory** (so the trigger is the value being *added*, not stored data); router has no short proxy timeout for media-api |
| — | **Still failing in production. UNRESOLVED — fix tracked in `lucas42/lucos_media_metadata_api#278`.** |

---

## Analysis

### Root cause — #274 added a synchronous eolas dependency to the save hot path

`lucas42/lucos_media_metadata_api#274` made `composer`/`producer` `ValueShapeURIObject` and wired **both** resolvers into their predicate `Config`: `ResolveNameToURI` (create-on-the-fly: `ResolveOrCreateEolasEntityByName("person", …)`) and `ResolveURIToName` (`ResolveEolasEntityName` → `fetchEolasName`). `updateTagsV3` / `updateTagsV3IfMissing` (`api/tracks_handler.go`) invoke these **synchronously, in the request path**, guarded by `if config.ResolveNameToURI != nil` / `if config.ResolveURIToName != nil`. Pre-#274 composer/producer were freetext literals with **no** eolas call on save. So #274 made saving a track that touches composer/producer block on eolas.

This is unique to composer/producer — the other eolas-URI predicates (offence, theme_tune, soundtrack, and memory as proposed in `lucas42/lucos_media_metadata_api#277`) are "link-only": they wire **no** resolver, so their saves make zero eolas calls. composer/producer are the lone outlier because they added create-on-the-fly resolvers.

> **Note (added on refinement):** the outlier status is **intentional, not an oversight** — lucas42 chose **inline-create** for composer/producer (`lucas42/lucos_media_metadata_manager#309`, enabled by `lucas42/lucos_search_component#176`): type a name → "Add…" → the Person is created in eolas on save. So "make composer/producer link-only like the others" (Direction 2 in the original Follow-up actions) is **ruled out** — it would undo that product decision. The durable fix instead keeps create-on-the-fly but makes the eolas dependency resilient (see the architect's design on `#278`).

### Contributing factor — eolas's bulk endpoint is slow

eolas's `/metadata/all/data/` endpoint is ~23s (full-dataset aggregation; `lucas42/lucos_eolas#283`). Notably, the URI→name resolver path `fetchEolasName(uri)` is implemented as `fetchEolasNames([uri])` — it fetches the **entire eolas dataset** to resolve a single name, so that sub-path inherits the 23s latency. eolas's per-entity paths are fast (create 43ms, list 0.22s); only the bulk aggregation is slow. The slowness may have been aggravated by #274 adding ~816 Person entities to that payload (unconfirmed — raised in #283).

### Why the 502 returns quickly (timing nuance)

The 502s were observed returning **fast** (same-second / a few seconds), which does **not** fit a clean 23s hang against the router's timeout — the `lucos_router` config has **no short proxy timeout** for media-api (nginx default ~60s, plain `proxy_pass`). The most likely explanation is that the **manager's** HTTP-client/PHP-FPM timeout (shorter than the API's hang) surfaces the 502 to the user while the API is still blocked on the eolas resolve. The **exact resolve sub-path** that hangs (URI→name bulk-fetch vs new-name create) was **not definitively isolated from logs**; a controlled production reproduction was available but **declined as unnecessary** — `#278` hardens the save-path eolas call regardless of which sub-path is involved, so isolating it would not change the fix, and the repro would have mutated production and added load to the already-degraded eolas.

### Detection / framing factor — the manager mislabels the status

The manager renders "Error updating track in API — API returned unexpected status code 400" on its error page, but the **real upstream status is 502**. The "400" is static manager text, not the actual response code — it sent the initial diagnosis chasing a validation-400 path that did not exist. Tracked as a secondary observation (below).

---

## What Was Tried That Didn't Work

- **Hypothesis: transient eolas window that self-recovered.** Wrong — the retry showed it persistent. (This was the original report's framing; corrected here.)
- **Hypothesis: clean 400 validation rejection** (manager posting name-only post-#274). Wrong — real status is 502; the "400" is a manager display mislabel.
- **Hypothesis: API panic / bad data in 22829.** Ruled out — no panic in raw logs; 22829 has no stored composer/producer/memory; API process stays healthy and serves other tracks throughout.
- **Hypothesis: 23s bulk-fetch hang hitting the router timeout.** Doesn't fit the fast-502 timing (router has no short timeout) — the manager-side timeout is the more likely 502 surface.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Make the composer/producer save-path eolas dependency resilient — design now posted on `#278` by the architect: **L1** drop the save-path eolas timeout 30s→2–3s (and make `fetchEolasName` a single-entity lookup, not a full bulk-dataset fetch); **L2** make URI→name backfill non-fatal (store URI, let reconcile converge the name); **L3** the create-on-the-fly fail-fast-vs-defer trade-off flagged for lucas42. ("Drop create-on-the-fly / link-only" is **ruled out** — conflicts with the inline-create product decision in `lucas42/lucos_media_metadata_manager#309`.) | `lucas42/lucos_media_metadata_api#278` | Open (design posted) |
| eolas `/metadata/all/data/` bulk-endpoint slowness (~23s; also no-ops the `reconcile_tag_names` job; `fetchEolasName` resolves a single URI through it) | `lucas42/lucos_eolas#283` | Open (triaged) |
| Manager renders "status code 400" when the real upstream status is 502 — should surface the actual status | To be ticketed by team-lead on `lucos_media_metadata_manager` | Pending |
| Confirm 22829 (and composer/producer saves generally) succeed once #278/#283 land | This report | UNRESOLVED — pending fix |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

---

## Appendix — note on diagnosis conditions

This incident was diagnosed during a session affected by intermittent tool-output corruption in the agent sandbox (`lucas42/lucos_agent_coding_sandbox#84`; harness argument on `lucas42/lucos#155`). One concrete downstream consequence: a garbled read caused a filed issue number to be mis-relayed (`lucos_eolas#89` instead of the correct `#283`), which had to be caught and corrected before it sent downstream work to the wrong issue. Cited here as real-world evidence of the impact of that class of fault.
