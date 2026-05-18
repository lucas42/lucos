# Incident: lucos_arachne ingest cron crashed on new owl:AsymmetricProperty rdf:type

| Field | Value |
|---|---|
| **Date** | 2026-05-17 / 2026-05-18 |
| **Duration** | ~14h 31m (2026-05-17 21:58Z first failure / 2026-05-18 11:23Z first alert → 2026-05-18 12:29:39Z monitoringRecovery) |
| **Severity** | Partial degradation (scheduled ingest cron failing; arachne search/SPARQL APIs themselves healthy) |
| **Services affected** | lucos_arachne (scheduled `lucos_eolas` ingest), downstream search index staleness for eolas data |
| **Detected by** | monitoring alert (`lucos_eolas` check on lucos_arachne, fired 2026-05-18 11:23:38Z after the threshold of 2-consecutive-failures was reached) |

---

## Summary

A new ontology predicate declaration in `lucos_eolas` introduced an OWL meta-type (`owl:AsymmetricProperty`) that the `lucos_arachne` search-index ingestor's hardcoded `IGNORE_TYPES` denylist didn't cover. Every subsequent run of the scheduled `lucos_eolas` ingest cron raised a `ValueError`, leaving the eolas-sourced data in the search index stale for ~14 hours. Resolved by a one-line addition to `IGNORE_TYPES` in lucas42/lucos_arachne#543. A follow-up replaces the fragile denylist with a namespace-based filter.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-17 21:53 | `lucos_eolas#256` merged — declares `eolas:preferredIdentifier` predicate with `rdf:type owl:AsymmetricProperty, owl:ObjectProperty` and a `skos:prefLabel`. |
| 2026-05-17 21:58 | lucos_eolas v1.0.56 deployed to avalon. |
| 2026-05-17 22:xx onwards | Successive scheduled `lucos_eolas` ingest runs after the deploy all error. Loganne records `knowledgeIngest: Knowledge graph partially updated — some sources failed`. No monitoring alert yet (threshold requires 2 of the most recent 2 to have failed *and* the most recent to be older than the schedule's freshness threshold). |
| 2026-05-18 11:23:38 | `monitoringAlert: 1 failing check on lucos arachne (arachne.l42.eu): lucos_eolas` fires. Debug field reports the last 3 scheduled-job runs errored; latest message identifies `<owl:AsymmetricProperty>` as the missing label. |
| 2026-05-18 11:26:32, 11:27:38 | Alert re-fires on subsequent monitoring polls. |
| 2026-05-18 ~12:00 | lucas42 raises the alert with SRE, providing the link to `lucos_eolas#256`. |
| 2026-05-18 ~12:15 | SRE confirms root cause (denylist gap), opens `lucas42/lucos_arachne#543` with a one-line `IGNORE_TYPES` addition. |
| 2026-05-18 12:21:35 | PR #543 merged. |
| 2026-05-18 12:27:20 | lucos_arachne v1.0.125 deployed to avalon (the fix). |
| 2026-05-18 12:29:35 | Post-startup auto-ingest emits `knowledgeIngest: Knowledge graph updated` — full success, no longer "partially updated — some sources failed". |
| 2026-05-18 12:29:39 | `monitoringRecovery: All checks healthy on lucos arachne`. Incident closed. |

---

## Analysis

### Root cause: an incomplete denylist met a fresh OWL meta-type

`ingestor/searchindex.py:graph_to_typesense_docs` iterates each subject and looks up its `rdf:type`. A hardcoded `IGNORE_TYPES` list lets it skip meta-types (predicate definitions, class definitions, ontology metadata) that shouldn't be indexed as items. Pre-incident it contained 9 entries — `owl:ObjectProperty`, `owl:Class`, `rdfs:Class`, `owl:DatatypeProperty`, `owl:Ontology`, `owl:TransitiveProperty`, `eolas:Category`, `rdf:Property`.

`lucos_eolas#256` declared `eolas:preferredIdentifier` with two rdf:types (`owl:ObjectProperty` and `owl:AsymmetricProperty`) and a `skos:prefLabel "preferred identifier"@en`. The `skos:prefLabel` made the predicate look like an indexable subject to the ingestor's loop. `owl:ObjectProperty` was in the denylist (good), but `owl:AsymmetricProperty` was not. When rdflib's non-deterministic iteration visited `owl:AsymmetricProperty`, the ingestor called `get_label()` on it, found no `skos:prefLabel` in the source RDF (correctly — OWL's own vocabulary isn't exported by eolas), and raised the strict error introduced by `lucas42/lucos_arachne#371` (which had removed the triplestore-fallback safety net for missing type metadata).

### Why this was missed pre-deploy

`lucos_eolas#256`'s unit test (`test_ontology_includes_preferred_identifier`) verified the predicate appears in the Turtle response with the correct `rdf:type`. There was no end-to-end test against the arachne ingestor, so the new combination of `(skos:prefLabel on a subject) + (only `rdf:type`s are OWL meta-types) + (one of those meta-types not in the denylist)` was never exercised before production.

The convention from `lucas42/lucos_arachne#371` had been ratified into `lucos_arachne/CLAUDE.md` ("sources must include type metadata for every `rdf:type` they emit"), but the convention is correct only for *domain* types — there's no realistic way for sources to provide metadata for OWL's own infrastructure terms. The mismatch between "what the convention says" and "what the ingestor's denylist actually filters" was the latent gap that this incident exposed.

### Why the alert took ~13 hours to fire

The monitoring check on `lucos_arachne/lucos_eolas` fails only when neither of the two most-recent runs of the tracked job succeeded — deliberate flap-suppression for scheduled jobs. The arachne ingestor's main scheduled run is `15 04 * * *` (daily at 04:15Z), and webhook-driven post-ingest updates from eolas count as additional runs. The cumulative failure pattern took roughly 13 hours from the buggy eolas deploy at 21:58Z to trip the threshold and produce the 11:23Z alert.

This delay is the *intended* behaviour for non-urgent scheduled-job failures and isn't something to change. The cost was acceptable here — search-index data for eolas was stale by 14 hours, but the live arachne SPARQL endpoint, search API, and webhook-driven updates from other sources all remained healthy throughout.

---

## What Was Tried That Didn't Work

Nothing this time — root cause was clear from the monitoring debug field (which gave the exact missing URI) plus a quick read of `searchindex.py:15-24`. The fix was identified within minutes of opening the file.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add `owl:AsymmetricProperty` to `IGNORE_TYPES` (hotfix) | lucas42/lucos_arachne#543 | Done (merged & deployed 2026-05-18 12:27Z) |
| Replace denylist with namespace-based meta-type filter; cover OWL/RDFS properties and classes; add direct test coverage for the meta-type-only skip path | lucas42/lucos_arachne#544 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
