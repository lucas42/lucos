# Incident: Track save shows 502 when an Album is selected in the `about` field

| Field | Value |
|---|---|
| **Date** | 2026-05-29 |
| **Duration** | First reported ~15:11 UTC; root cause verified ~16:50 UTC. User had a workaround (avoid the `about` field); fixes tracked, not yet shipped. |
| **Severity** | Partial degradation — user-blocking for track saves that set `about`/`mentions` to a non-eolas entity |
| **Services affected** | lucos_media_metadata_manager (save UX + error rendering), lucos_media_metadata_api (validation + logging) |
| **Detected by** | User report — lucas42, via team-lead (blocked saving track 22829) |

> **Correction history.** This report was published twice with a **wrong** root cause before this version. The first two versions (`lucas42/lucos#202`, corrected by `#203`, refined by `#204`) blamed a synchronous eolas-resolution call on the composer/producer save path added by `lucas42/lucos_media_metadata_api#274`. **That was wrong, and — more importantly — it was never verified, only inferred.** This version is the verified cause, reproduced. See "What we got wrong and why" below; that section is the most valuable part of this report.

---

## Summary

Saving a track after selecting an **Album** in the **`about`** field showed an HTTP **502** error page reading "Error updating track in API — API returned unexpected status code 400". The real chain: the manager's `about` field is an unscoped entity-search that offers Albums (media-metadata-origin URIs); the API correctly **rejects** that with a clean **HTTP 400** (the `about` predicate only allows eolas-origin URIs); the API **logs nothing** about the rejection; and the manager's save controller catches *any* API error and renders the page with a **hardcoded 502** (embedding the real "400" only as message text). So: real status 400, user sees 502, and the server side is silent. Three distinct bugs, none involving composer/producer or eolas latency. The user's workaround is to avoid the `about` field. Fixes tracked in `lucas42/lucos_media_metadata_api#279`, `lucas42/lucos_media_metadata_manager#312`, and `lucas42/lucos_media_metadata_manager#311`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-29 ~15:11 | lucas42 reports saving track 22829 fails — 502 page whose body says "status code 400" |
| 15:11–16:00 | Investigation publishes (and twice re-publishes) a composer/producer-eolas root cause — **inferred, never reproduced** |
| ~16:30 | lucas42 clarifies: he did **not** touch composer/producer; he selected an **Album** in the **`about`** field; failure pre-dates today's changes (latent) |
| ~16:47 | Controlled repro on a throwaway test track: `PATCH about` = Album-origin URI → **HTTP 400, 67ms**, origin-validation; eolas-origin control → 200; the 400 is **not logged** |
| ~16:50 | Manager code review confirms `updateTrack` renders any `ApiError` as a hardcoded 502; `about`/`mentions` form fields confirmed unscoped. Verified chain complete |
| — | Workaround in place (avoid `about`). Fixes tracked; not yet shipped |

---

## Analysis

The failure is the coincidence of three independent defects. None is in the composer/producer path or eolas.

### Bug 1 — `about`/`mentions` search fields are unscoped (manager)

In `lucos_media_metadata_manager/src/formfields.php`, `about` and `mentions` are `"type" => "search"` with **no `"types"` scope** — unlike `theme_tune` (`"types" => "Creative Work"`), `offence` (`"Offence"`), `language` (`"Language"`). An unscoped entity-search surfaces **all** indexed entities, including media-metadata-origin **Albums** (verified: arachne search for an album title returns `[Album]` results with `https://media-metadata.l42.eu/albums/N` URIs). So the UI offers the user a value the API will always reject. Tracked: `lucas42/lucos_media_metadata_manager#312`.

### Bug 2 — the API rejects it (correctly) but logs nothing (api)

The `about` predicate has `AllowedOrigins: [eolas]`. A `media-metadata.l42.eu/albums/N` URI fails `ValidateURIOrigin`, so `updateTagsV3` returns a 400. **Reproduced** on a throwaway track: `HTTP 400, 67ms`, body `{"error":"uri \"https://media-metadata.l42.eu/albums/2923\" does not start with an allowed origin [https://eolas.l42.eu]","code":"invalid_tag_value","predicate":"about"}`; the eolas-origin control returned 200. The rejection is **correct behaviour** — but the API writes **no log line** recording it (verified: only `track update request` / `update/create track` appear). That silence is why the incident was invisible server-side and why the first diagnosis went unchecked. This is the highest-value systemic fix. Tracked: `lucas42/lucos_media_metadata_api#279`.

### Bug 3 — the manager manufactures a 502 from any API error (manager)

`lucos_media_metadata_manager/src/controllers/updatetrack.php`:

```php
try {
    fetchFromApi("/v3/tracks/{$trackid}", "PATCH", $api_data);
    header("Location: /tracks/{$trackid}?saved=true", true, 303);
} catch (ApiError $error) {
    displayError(502, "Error updating track in API.\n\n".$error->getMessage());
}
```

`fetchFromApi` (`src/api.php`) throws `ApiError("API returned unexpected status code {$code}")` for any response ≥300; the catch renders the page with a **hardcoded `http_response_code(502)`** (`displayError` sets exactly the status passed). So a real **400** becomes a **502** page whose body text says "status code 400". Tracked: `lucas42/lucos_media_metadata_manager#311` (being re-framed by team-lead from "cosmetic mislabel" to "confirmed 502 source").

### Verification status (how we know this is the actual cause, this time)

- Bugs 1 and 2 were **reproduced** directly against a throwaway test track (created, exercised, deleted — DELETE 204 / GET 404).
- Bug 3 (the manager 502) was **not** driven through the CSRF-gated form end-to-end, but is established by **code + an exact symptom fingerprint**: a 502 page whose *body* says "API returned unexpected status code 400" is uniquely produced by `displayError(502, "…".$error->getMessage())` where `getMessage()` is "…status code 400" — nothing else in the system yields that specific combination. So the actual failing request demonstrably took this path; a CSRF-driven end-to-end would only re-confirm the fingerprint at real cost.

---

## What we got wrong and why

The first two published versions named a **composer/producer eolas-resolution** root cause that was **inferred, never reproduced** — and it was wrong. The real cause is unrelated (`about`/Album) and latent (pre-dating today's migration). Every warning sign was present and rationalised past:

- **The blamed thing wasn't on the affected record.** Track 22829 had **no composer/producer tag** — yet the diagnosis blamed the composer/producer save path. (A tag the affected item doesn't have cannot be what failed.)
- **The diagnosis hinged on an unconfirmed assumption about what the user was doing** ("he must have been setting a composer/producer") — never confirmed with him. He was setting `about`.
- **The timing didn't fit and was rationalised.** The observed 502 returned fast; the hypothesised path was a slow synchronous eolas call. That mismatch was explained away (manager timeout) rather than treated as disproof.
- **A controlled reproduction was offered and declined** as "unnecessary, the fix helps regardless" — the one step that would have caught the error.
- **The cause was inferred chiefly from recency** ("a migration touched this area today"), which crowded out the simpler, latent explanation.

The corrective process change is recorded in `references/incident-reporting.md` § "Verify the root cause actually caused *this* failure". The lesson: a plausible, well-reasoned mechanism is **not** a root cause until it's confirmed it produced *this* failure — by reproduction or direct evidence the failing request took that path. Naming an unverified mechanism as the cause ended the investigation prematurely and sent fix effort at the wrong target across three PRs.

---

## What Was Tried That Didn't Work

- **Composer/producer eolas-resolution hypothesis** (versions 1–2): wrong; the track had no such tag and the path never fired. Disproved by reproduction.
- **"Transient eolas window that recovered"** (version 1): wrong; the failure was persistent.
- **Direct-to-API repro to reproduce the 502**: produced a clean 400, not a 502 — correctly redirecting attention to the manager layer as the 502 source.
- **A `curl`-through-the-gateway test that returned 200-no-change**: a test artifact (the local curl didn't forward the body); unrelated to the real failure. Ruled out, not chased.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| API: log track-save validation rejections (currently silent) — highest-value systemic fix | `lucas42/lucos_media_metadata_api#279` | Open |
| Manager: scope the `about`/`mentions` search fields so the UI can't offer non-eolas entities the API always rejects | `lucas42/lucos_media_metadata_manager#312` | Open |
| Manager: stop rendering all API errors as a hardcoded 502 with the wrong status in the text — surface the real upstream status | `lucas42/lucos_media_metadata_manager#311` | Open (being re-framed as the confirmed 502 source) |
| Process: verify a root cause actually caused *this* failure before publishing it | `references/incident-reporting.md` (new section, cites this incident) | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

---

## Appendix — note on diagnosis conditions

Early diagnosis was also degraded by intermittent tool-output corruption in the agent sandbox (`lucas42/lucos_agent_coding_sandbox#84`; harness argument on `lucas42/lucos#155`). One concrete consequence: a garbled read caused a filed issue number to be mis-relayed (`lucos_eolas#89` instead of the correct `#283`), caught and corrected before it sent downstream work astray. The wrong *root cause*, however, was not a tooling artifact — it was an inference-without-verification failure, addressed by the process change above.
