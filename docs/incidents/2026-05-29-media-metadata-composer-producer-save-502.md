# Incident: Track saves 502 when eolas is slow (composer/producer save-path dependency)

| Field | Value |
|---|---|
| **Date** | 2026-05-29 |
| **Duration** | observed ~15:07–15:11 UTC on track 22829; underlying fragility ongoing until `lucas42/lucos_media_metadata_api#278` ships |
| **Severity** | Partial degradation (user-blocking for composer/producer track edits while eolas is slow) |
| **Services affected** | lucos_media_metadata_api (save path), lucos_media_metadata_manager (save UX), lucos_eolas (degraded dependency) |
| **Detected by** | User report — lucas42, via team-lead (blocked saving track 22829) |

---

## Summary

lucas42 was blocked saving track 22829 in the media-metadata manager: the save returned an error ("API returned unexpected status code 400", and in the manager access log a **502 Bad Gateway**). Root cause: PR `lucas42/lucos_media_metadata_api#274` (the composer/producer → `eolas:Person` migration) added a **synchronous eolas call to the track-save hot path** for composer/producer tags. Around 15:07–15:11 UTC eolas was degraded (its bulk `/metadata/all/data/` endpoint at ~23s, timing out), coinciding with a 1.0.63 redeploy whose startup `reconcileTagNames` run hammered that same slow endpoint — so a save touching composer/producer hung on the eolas call and the gateway returned 502 (and, when eolas erred rather than hung, the API returned 400). eolas's fast paths recovered shortly after (create 43ms, list 0.22s). **Resolution confirmation is pending a user retry of the 22829 save — TBD.** The durable fix (remove the synchronous eolas dependency from the save path) is tracked in `lucas42/lucos_media_metadata_api#278`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~13:00–15:00 | (Earlier) lucas42 sees "API returned unexpected status code 400" saving 22829 — the eolas-errored face of the same cause |
| 15:03 | `lucas42/lucos_media_metadata_api#276` (run `reconcileTagNames` on startup; fix for #275) merged |
| 15:07:27, 15:07:38 | Manager → API health probe returns **502** (API down during redeploy) |
| 15:07:40 | API container re-created → **1.0.63** deployed |
| 15:07:41 | `reconcileTagNames` runs on startup (the #276 fix), begins bulk eolas fetch of 2517 URIs from `/metadata/all/data/` |
| 15:08:11 | Bulk eolas fetch **times out at 30s** (`context deadline exceeded`, `resolved=0`) |
| 15:11:32 / :39 / :48 | lucas42's `PATCH /v3/tracks/22829` → API logs `update/create track` then **no completion / no panic**; manager access log shows **502** for each |
| ~15:21 | eolas fast paths confirmed healthy (create endpoint 43ms, person-list 0.22s); bulk `/metadata/all/data/` still ~23s |
| TBD | lucas42 retries 22829 save → confirms transient recovery, **or** surfaces a track-22829-data-specific cause to investigate |

---

## Analysis

### Stage 1 — Migration introduced a synchronous eolas dependency on the save hot path

`lucas42/lucos_media_metadata_api#274` made `composer`/`producer` `ValueShapeURIObject` and wired **both** resolvers into their predicate `Config`: `ResolveNameToURI = resolveOrCreateEolasEntityHTTP` (create-on-the-fly) and `ResolveURIToName`. `updateTagsV3` / `updateTagsV3IfMissing` (`api/tracks_handler.go`) call these **synchronously in the request path** (guarded by `if config.ResolveNameToURI != nil` / `if config.ResolveURIToName != nil`), with a 30s HTTP client timeout. Pre-#274 composer/producer were freetext literals with **no** eolas call on save. So saving a track that touches composer/producer now blocks on eolas. (The other eolas-URI predicates — offence, theme_tune, soundtrack, and memory as proposed in #277 — are "link-only": they wire no resolver and make zero eolas calls on save. composer/producer are the outlier.)

### Stage 2 — eolas was degraded on its bulk endpoint

eolas's `/metadata/all/data/` endpoint (2517 URIs) measured ~23s and was timing out at the 30s client deadline. eolas's other paths were fast (create 43ms, list 0.22s), so the degradation was endpoint-specific, not a full eolas outage. The bulk-endpoint slowness may have been aggravated by #274 adding 816 new Person entities to that payload — **unconfirmed**.

### Stage 3 — Redeploy + reconcile-on-startup coincided with the user's edit

The 15:07:40 redeploy to 1.0.63 shipped `#276` (run `reconcileTagNames` on startup, the fix for the deploy-starvation issue `#275`). A correct fix in isolation — but it means every startup now fires the heavy 2517-URI bulk-eolas fetch, which hung for 30s at 15:07:41–15:08:11 and loaded the already-slow eolas. lucas42's saves at 15:11 fell in/just after this degraded window, so the save-path eolas call hung → gateway 502.

### Detection / framing factors

- The API logs `update/create track` but **does not log the save-path eolas failure reason** at a visible level — the request just goes silent, which slowed diagnosis.
- The same cause presents two faces: **400** ("could not resolve…") when the eolas call errors, **502** when it hangs past the gateway timeout. The initial report was "status code 400"; the live logs showed 502. Recognising these as one cause (not two bugs) was key.

---

## What Was Tried That Didn't Work

- **Initial hypothesis: a clean 400 `requires_uri` validation** from the manager posting name-only composer/producer post-#274 (manager-side #309 not yet shipped). The manager access log showed **502**, not 400 — redirecting the investigation from "validation rejection" to "gateway/hang".
- **Hypothesis: API panic on track 22829's data.** Ruled out — raw unfiltered API logs (including non-timestamped stack frames) showed no panic after `update/create track`.
- **Hypothesis: eolas fully down.** Ruled out — eolas create (43ms) and list (0.22s) were fast; only the bulk endpoint (23s) was degraded.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Remove the synchronous eolas dependency from the composer/producer save path (short timeout + accept-name-and-reconcile-later, or drop create-on-the-fly for URI-via-search, or async) — design routed to architect | `lucas42/lucos_media_metadata_api#278` | Open |
| Investigate eolas `/metadata/all/data/` bulk-endpoint slowness (~23s, timing out — also no-ops the `reconcile_tag_names` job) | Pending — cross-repo routing decision with team-lead (offered to file on `lucas42/lucos_eolas`) | Pending |
| Confirm 22829 save succeeds post-recovery (user retry) | This report (resolution TBD) | Pending |

Note: the deploy-starvation issue `lucas42/lucos_media_metadata_api#275` and its fix `#276` are referenced here only because the #276 deploy/startup-reconcile was a contributing coincidence; #275/#276 are themselves resolved.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
