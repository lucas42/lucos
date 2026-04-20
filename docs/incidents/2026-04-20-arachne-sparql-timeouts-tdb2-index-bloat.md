# Incident: Arachne SPARQL timeouts from TDB2 index bloat

| Field | Value |
|---|---|
| **Date** | 2026-04-20 (diagnosed and resolved) |
| **Duration** | Chronic degradation from 2026-04-08 to 2026-04-20 (~12 days); acute resolution took ~10 minutes once the root cause was identified |
| **Severity** | Partial degradation — intermittent SPARQL query timeouts, no hard outage |
| **Services affected** | `lucos_arachne` (triplestore, explore UI, MCP server, ingestor), and any estate-wide consumer of `/arachne/sparql` |
| **Detected by** | User report ("feels as unreliable as it used to be") |

Source: no dedicated critical issue — the user asked SRE to investigate directly. Resolution tracked in lucas42/lucos_arachne#387 (memory bump, merged) and lucas42/lucos_arachne#386 (strategic follow-up, open with architect).

---

## Summary

On 2026-04-08 lucas42/lucos_arachne#268 replaced the OWL reasoner with a pre-computed `urn:lucos:inferred` graph, and — on the same change — dropped the triplestore container memory limit 2G → 1G and JVM `-Xmx` 1600m → 768m. The sizing was done on the expectation that removing the in-memory OWL reasoner would net-save RAM; in practice the new ingestion pattern (`DROP GRAPH` + re-`INSERT` on every run, ≈2×/day) produced TDB2 B+tree tombstones that were never reclaimed. Over ~12 days the six quad indexes grew from ~200MB-worth of live data to **~93GB of mostly-dead pages** — far larger than the container's total memory budget — so the JVM started swapping and every query hit disk. Users perceived this as intermittent SPARQL timeouts; two earlier symptom-level tickets (lucas42/lucos_arachne#321 list_types 34s query and lucas42/lucos_arachne#343 triplestore healthcheck flapping) had been closed without identifying the underlying bloat. Online compaction of the TDB2 store (`POST /$/compact/arachne?deleteOld=true`) reclaimed 98GB of disk and restored `ASK {}` latency from 200+ ms to 3–6 ms in 50 seconds; a follow-up PR raised the container and heap limits to give ongoing headroom; and a separate open-ended issue captures the write-pattern redesign with the architect.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-08 ~14:34 | lucas42/lucos_arachne#268 merges. Container memory 2G → 1G, `-Xmx` 1600m → 768m, OWL reasoner replaced by pre-computed inferred graph. Every ingestion run from this point forward does `DROP GRAPH` + re-`INSERT` of all source graphs |
| 2026-04-08 onwards | ≈2 ingestion cycles/day accumulate TDB2 tombstones. Dataset on disk grows roughly 8GB/day while live quad count stays flat at ~227K |
| 2026-04-10 14:03 | `list_types` MCP query times out at 30s client side — actually takes 34.4s server-side. Filed as lucas42/lucos_arachne#321 |
| 2026-04-10 14:28 | lucas42/lucos_arachne#322 merged: client read timeout bumped 30s → 60s. Root cause (GC/swap pressure, not dataset size) not investigated |
| 2026-04-13–14 | Triplestore healthcheck flaps 6 times in 24h — each alert ≥70s of unresponsiveness. Filed as lucas42/lucos_arachne#343 |
| 2026-04-14 13:34 | lucas42/lucos_arachne#343 closed after a hand-wave attribution to "ingestor bulk writes blocking healthcheck" — no fix applied, no follow-up |
| 2026-04-20 14:28–14:32 | Live Fuseki log inspection shows `ASK {}` healthcheck latency drifting 2ms → 200+ ms within an 8-minute window under no unusual load |
| 2026-04-20 14:33 | Container memory observed at **1016 MiB / 1 GiB (99.23%)**. `/proc/<java>/status` shows `VmSwap: 485 MB` — JVM heap itself is being swapped out |
| 2026-04-20 14:40 | TDB2 data directory measured at **~93 GB on disk** against 227,499 live quads. Root cause identified: un-reclaimed B+tree tombstones from repeated `DROP GRAPH` + re-`INSERT` |
| 2026-04-20 14:43:21 | `POST /$/compact/arachne?deleteOld=true` triggered. Online compaction task 1 starts, builds `Data-0002-tmp` alongside `Data-0001` |
| 2026-04-20 14:44:11 | Compaction completes successfully in 50s. `Data-0001` removed; `Data-0002` is **76 MB** (1225× smaller). Triple count preserved (227,489). Host disk reclaimed 98 GB |
| 2026-04-20 14:44– | Post-compaction `ASK {}` latency 3–6 ms. All monitoring checks green |
| 2026-04-20 14:44 | lucas42/lucos_arachne#386 opened — open-ended issue on long-term ingestion strategy, routed to the architect |
| 2026-04-20 14:48 | lucas42/lucos_arachne#387 opened — follow-up PR raising container memory 1G → 2G and `-Xmx` 768m → 1024m |
| 2026-04-20 ~14:52 | lucas42/lucos_arachne#387 approved by lucos-code-reviewer; auto-merge triggered, CI running |
| 2026-04-20 ~15:05 | Architect replies on #386: recommended order is **conditional refresh → diff-based ingestion → scheduled compaction as belt-and-braces**; offered to draft ADR-0002 once direction is agreed |

---

## Analysis

### Factor 1: memory sizing change based on an incomplete mental model

lucas42/lucos_arachne#268 framed itself as a net memory reduction: removing the `OWLMicroFBRuleReasoner` in-memory model "saved" ~500 MB, so container memory and JVM heap could both drop. The first half of that reasoning was correct — the reasoner was genuinely gone. The second half missed that the replacement pre-computed `urn:lucos:inferred` graph isn't an in-memory object at all; it's persisted in the same TDB2 store as the raw data, and its indexes have to be memory-mapped by the JVM process at query time. The container's memory ceiling has to accommodate JVM heap + direct buffers + every TDB2 index file the kernel decides to keep resident, and after the change there was essentially no budget left for the last category.

This is a failure mode worth naming: when refactoring *where* work happens, check whether the memory that was implicit in the old design has just moved rather than been freed. "Pre-computed" doesn't mean "free" — it means "paid at write time and stored somewhere."

### Factor 2: TDB2's delete semantics make full-refresh ingestion quietly pathological

TDB2 B+tree indexes mark deleted pages as tombstoned and never reclaim the space without an explicit compaction. The ingestor's strategy is a full `DROP GRAPH` + re-`INSERT` on every run — for all five source graphs and the inferred graph — ≈2 runs per day. Each run tombstones ~227K quads × 6 quad indexes and then inserts ~227K fresh quads. After ~80 cycles the indexes were ~4 orders of magnitude larger than the live data.

This isn't a Jena bug; it's documented behaviour. But it's also not a storage pattern most tools expect when you write "just rewrite the dataset periodically," which is why the ingestor design flew through review. Tracked for strategic redesign in lucas42/lucos_arachne#386.

### Factor 3: two earlier symptom tickets closed without root-cause

lucas42/lucos_arachne#321 (slow list_types) and lucas42/lucos_arachne#343 (healthcheck flapping) were both closed within hours of opening, each with a local symptom fix: #321 bumped the client timeout from 30s to 60s; #343 concluded that "ingestor bulk writes probably blocked the healthcheck" and took no action. Neither investigated why the query had become slow *now*, or why the healthcheck had started flapping *now*, when nothing had visibly changed in either component. The "nothing changed" instinct was wrong — the TDB2 store had been slowly poisoning itself since #268 shipped, and both symptoms were downstream of the same cause.

If the "what changed recently?" question had been asked against an `ls -lah /fuseki/run/databases/arachne/Data-0001/` baseline, the index growth would have been visible immediately.

### Factor 4: monitoring blind to progressive degradation

The triplestore's `ASK {}` healthcheck is cheap enough that it still completed in 2–4 ms most of the time even while larger queries were timing out. The flapping in #343 was the only signal from monitoring, and it was sporadic. User-reported SPARQL timeouts are the class of signal we currently have no direct monitoring for — the `fetch-info` check only covers whether `/_info` returns a 200, not whether downstream SPARQL queries are fast enough to be useful. This made the degradation effectively invisible to monitoring for ~12 days.

---

## What Was Tried That Didn't Work

Prior attempts on symptom tickets (both made by other agents before today's investigation):

- **lucas42/lucos_arachne#322 — bump `list_types` client timeout 30s → 60s.** Stopped the MCP tool erroring, but hid the underlying slowness. The same query today runs in sub-second against the compacted store, so the 60s is now pure headroom — a retrospective overcorrection rather than a fix.
- **lucas42/lucos_arachne#343 — attribute healthcheck flapping to "ingestor blocking Fuseki."** The conclusion was plausible but untested. Today's Fuseki log inspection shows healthcheck latency drifting under *no* concurrent ingest activity, so the hypothesis doesn't hold. The real driver was GC pressure and page-cache eviction from the over-saturated container, not write-lock contention.

During today's investigation itself, everything worked on the first attempt — the initial hypothesis (memory pressure from TDB2 bloat) matched the on-disk evidence and the compaction confirmed it by reclaiming 98 GB.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Online compaction of the TDB2 store to reclaim tombstoned index pages | — (ad-hoc `POST /$/compact/arachne`) | Done |
| Raise triplestore container memory 1G → 2G and JVM `-Xmx` 768m → 1024m to restore headroom for TDB2 mmap and JVM overhead | lucas42/lucos_arachne#387 | Merged |
| Decide long-term ingestion strategy (conditional-refresh, diff-based, scheduled compaction, etc.) — architect input requested | lucas42/lucos_arachne#386 | Open |
| Author ADR-0002 documenting the chosen ingestion strategy (architect offered once direction is agreed) | to be raised after #386 resolves | Pending |
| Add a direct SPARQL-latency signal to the `/_info` monitoring so progressive query degradation is visible before users report it | lucas42/lucos_arachne#388 | Open |
| Add periodic TDB2 compaction to the ingestor or a cron, as belt-and-braces until the write-pattern redesign lands | lucas42/lucos_arachne#389 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
