# Incident: Eolas LoC URI webhooks broke arachne ingestion and stranded loganne

| Field | Value |
|---|---|
| **Date** | 2026-05-07 |
| **Duration** | ~62 minutes (22:43 UTC to 23:45 UTC) |
| **Severity** | Partial degradation / Security |
| **Services affected** | lucos_arachne (brief), lucos_loganne (sustained), lucos_eolas (data not propagating) |
| **Detected by** | monitoring alerts (`arachne ingestor`, `loganne webhook-error-rate`) |

---

## Summary

A routine `python manage.py load_language_families` run on lucos_eolas to add the `zxx` no-linguistic-content language (lucas42/lucos_eolas#235) emitted 117 `itemUpdated` webhook events in 58 seconds. Each event's `url` field carried the canonical Library of Congress URI for the language family (e.g. `http://id.loc.gov/vocabulary/iso639-5/cau`) rather than an eolas-served URL. When loganne fanned the events out to arachne's webhook handler, arachne fetched LoC's JSON-LD instead of eolas's RDF — which uses LoC vocabularies (`mads:Topic`, `mads:Language`, `skos-xl:Label`, etc.) without the type metadata required by lucas42/lucos_arachne#371. All 117 webhook deliveries failed and were stranded in loganne. The arachne `ingestor` check briefly alerted under burst load and recovered in 2 minutes. As a side effect, arachne's webhook handler sent the `KEY_LUCOS_EOLAS` Bearer token to `id.loc.gov` 42+ times. Resolved by shipping lucas42/lucos_eolas#238 (introducing `get_webhook_url()`), re-running `load_language_families` to emit corrected events, manually retrying a handful of burst-overload 504s, and editing `loganne` state to remove the 115 unrecoverable old events. Credential rotated by lucas42 in parallel.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 22:43:38 | `load_language_families` run starts on avalon; eolas begins emitting `itemUpdated` events with `url` = LoC URI |
| 22:44:36 | Last event of the burst (~117 events in 58 seconds) |
| 22:45:25 | `monitoringAlert`: arachne `ingestor` check failing (TCP connect to ingestor:8099) — burst overload |
| 22:47:26 | `monitoringRecovery`: arachne fully healthy again |
| 22:48 | `monitoringAlert`: loganne `webhook-error-rate` failing (115 of 117 webhook deliveries to arachne failed) |
| 22:48 | team-lead pings lucos-site-reliability with lucas42's hunch that the language-family load is the trigger |
| 22:50 | SRE confirms arachne now healthy; identifies stranded webhooks in loganne |
| 22:55 | SRE retries Gateway-Time-out events 1-by-1 with 1s pacing — every retry converts to a permanent validator failure (`Source RDF does not include a label for <X>`) — confirming the failures are structural, not load-related |
| 23:07 | lucas42/lucos_eolas#237 filed (P2): root cause analysis — webhook URLs point at id.loc.gov |
| 23:07 | lucas42/lucos_arachne#443 filed (P2/security): auth Bearer token leaks to non-lucos URLs |
| ~23:10 | lucas42 rotates `KEY_LUCOS_EOLAS` via lucos_creds; redeploys eolas and arachne |
| 23:22:51 | lucas42/lucos_eolas#238 (PR adding `get_webhook_url()`) merges |
| 23:27:13 | lucos_eolas v1.0.43 deployed to avalon — fix live |
| 23:40:23 | SRE re-runs `load_language_families` on avalon; new events emit with corrected eolas-hosted URLs |
| 23:40:50 | Re-run completes; 68 fresh `itemUpdated` events emitted (only changed records) |
| 23:41 | Burst causes 23 fresh 504s on arachne; loganne's auto-retry-once clears 21 of them; 2 remain stuck |
| 23:42 | SRE manually retries the 2 remaining new-burst 504s — both succeed |
| 23:43 | A further 4 new-burst 504s surface; SRE retries them with 1s pacing — all succeed. New events all delivered |
| 23:44:38 | Loganne stopped via `docker stop lucos_loganne` |
| 23:44:45 | Ephemeral `python:3-slim` container with the loganne state volume mounted: filter out the 115 LoC-URL UUIDs from `events.json` (7230 → 7115 events) |
| 23:44:48 | Loganne restarted via `docker start lucos_loganne` |
| 23:45:08 | Loganne reports healthy with `webhook-error-count: 0` |
| 23:45:21 | Monitoring confirms loganne healthy; estate fully healthy (51/51 systems green) |
| 23:46 | Verified arachne now contains language family entities via MCP find_entities query |

---

## Analysis

### Root cause: external canonical URI used as webhook fetch target

`LanguageFamily.get_absolute_url()` in eolas returned the canonical Library of Congress URI (`http://id.loc.gov/vocabulary/iso639-5/<code>`) for non-synthetic families. The post_save signal then sent that URL straight to loganne as the `url` field of the `itemUpdated` event. When loganne POSTed the event to `https://arachne.l42.eu/webhook`, arachne's `fetch_url(...)` (in `ingestor/server.py`) called the LoC URL directly — getting LoC's JSON-LD, which uses LoC's own type vocabularies and contains none of eolas's required type metadata. Arachne's lucas42/lucos_arachne#371 validator (which enforces self-contained RDF) then rejected every one of the 117 events.

The `qli` (language isolates) and `qsp` (ISO 639 special codes) synthetic families were unaffected — their `get_absolute_url` returns a `BASE_URL`-prefixed eolas-served URL, so arachne fetched eolas's own RDF for them.

The conceptual confusion was conflating **"the canonical identifier for an entity"** with **"the URL we want subscribers to fetch when they see this event."** For most lucos models, those are the same URL. For models whose canonical identifier is an external URI (LoC, Wikipedia, Wikidata, etc.), they must be different. The fix added a `get_webhook_url()` method that defaults to `get_absolute_url()` but can be overridden — `LanguageFamily` overrides it to return an eolas-hosted URL, while `get_absolute_url()` keeps returning the LoC URI as the canonical identifier.

### Contributing: arachne sends auth token to whatever URL the webhook payload says

`lucos_arachne/ingestor/authorised_fetch.py:12` decides whether to attach a `KEY_LUCOS_<SYSTEM>` Bearer token based only on the source system name in the webhook event, not on the destination URL's hostname. So arachne sent the eolas auth token to `id.loc.gov` 42+ times during the incident — a credential exposure to an externally-operated service.

The token was rotated by lucas42 in parallel with the fix. Tracked in lucas42/lucos_arachne#443.

### Contributing: loganne webhook auto-retry only once

When the initial burst overwhelmed arachne and produced 504s, loganne's once-only auto-retry policy (already documented in agent memory) gave up on most of them as permanent failures. Even after arachne recovered seconds later, the events were marked terminal `failure` and would never have been re-delivered without manual API calls.

This was the proximate reason `webhook-error-rate` stayed red — but the deeper issue was that even with infinite retries, the failures would have remained structural (LoC URL → wrong RDF), not transient.

### Contributing: no DELETE API on loganne events

Once 115 events were stranded with `url` pointing at `id.loc.gov`, no amount of retrying could clear them — retries re-deliver the stored payload, they don't re-fetch the URL from the source. With no DELETE API, the only ways to clear the alert were (a) wait 90 days for auto-trim, (b) manually edit `events.json` on the host, or (c) lose the alert visibility.

This is by design and will not be addressed: event deletion is intended to remain an exceptional, host-level operation, and exposing it via an API would reduce that friction in an undesirable way. The trade-off is accepted: when a class of misrouted-webhook stranding recurs, clearing it requires docker access on the loganne host (as in this incident's resolution) rather than an API call.

### Detection / triage

- The `arachne ingestor` alert at 22:45 was a useful signal but was a transient symptom rather than the actual problem; it self-recovered in 2 minutes.
- The `loganne webhook-error-rate` alert was the right indicator that something needed attention. It correctly stayed red until manually cleared.
- lucas42's hunch ("the language-family load might be the trigger") was correct on the trigger but framed the problem as load-related; the actual cause was structural URL semantics. Sampling the distribution of `e.webhooks.errorMessage` across the failure set early (per the new `feedback_sample_webhook_errors_first.md` memory rule) would have caught this within seconds had it been done before any retry attempts.

---

## What Was Tried That Didn't Work

- **Bulk retry via `POST /events/retry-webhooks`** at 22:52 timed out at the nginx gateway (`504 Gateway Time-out`) because firing 115 webhooks in parallel re-created the original burst against arachne. Only 2 cleared.
- **Per-UUID retry of all 113 events with 0.4s pacing** (background task) was correct in approach but stalled because each retry waited for the synchronous webhook delivery before responding. Stopped early after 1 retry and reassessed.
- **Initial diagnostic hypothesis** that the 504s were burst-overload and would clear on calm retry was wrong. Retrying all 42 "Gateway Time-out"-class events one at a time with 1s spacing showed every single one converted to a permanent validator failure (`Source RDF does not include a label for <X>`) — the LoC URL was the real problem, not load.
- **Sending `cc lucos-security` in the body of lucas42/lucos_arachne#443** for the credential-rotation referral did not actually notify lucos-security — agents don't watch GitHub mentions. team-lead messaged lucos-security directly. SRE persona file updated to flag this pattern.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add `get_webhook_url()` to eolas; use it in post_save signal; override on `LanguageFamily` | lucas42/lucos_eolas#237 → lucas42/lucos_eolas#238 | Done (merged 23:22, deployed 23:27) |
| Restrict arachne `authorised_fetch` to lucos-host destinations; reapply origin check on redirect | lucas42/lucos_arachne#443 → lucas42/lucos_arachne#444 | Done (merged 23:51) |
| Rotate `KEY_LUCOS_EOLAS` (leaked to id.loc.gov 42+ times) | (handled by lucas42 via lucos_creds Refresh) | Done |
| Add DELETE / expire API to loganne so misrouted events can be cleared without host-level intervention | n/a — declined | Won't fix. lucas42's call (2026-05-07): event deletion should remain an exceptional, host-level operation; making it API-accessible is undesirable friction reduction |
| SRE memory: sample webhook error distribution before bulk-retrying | `feedback_sample_webhook_errors_first.md` | Done |
| SRE persona: don't `cc` agents in issue bodies | `~/.claude/agents/lucos-site-reliability.md` (commit 46e3593) | Done |

---

## Sensitive Findings

[x] Yes — see note below.

The `KEY_LUCOS_EOLAS` Bearer token was leaked to `id.loc.gov` in HTTP `Authorization` headers an estimated 42+ times during the burst (one per event arachne attempted to fetch from the misrouted URL). The actual token value is not included in this report; the credential has been rotated by lucas42 via lucos_creds. The leak path (arachne's webhook handler sending Bearer tokens to whatever hostname the webhook URL specifies) is tracked for fix in lucas42/lucos_arachne#443 — that fix is required before this class of leak is structurally prevented; current containment is the rotation alone.
