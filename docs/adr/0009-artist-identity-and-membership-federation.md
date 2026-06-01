# ADR-0009: Artist identity and membership federation (media ↔ eolas ↔ arachne)

**Date:** 2026-06-01
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos_arachne/issues/597

## Context

`lucos` ADR-0005 settled the media-ecosystem URI namespace: media services store only their own URIs or `lucos_eolas` URIs, never `lucos_contacts` URIs directly. `lucos_media_metadata_api#246` settled that an Artist lives in media as `mo:MusicArtist` (minted from name, covering both individuals and groups so that **no person-vs-group decision is taken at ingest time**), with the artist-who-is-also-a-person case to be handled by an **optional, manually-curated `owl:sameAs`** to the corresponding `eolas:Person`, federated by `lucos_arachne` for display. `lucos_media_metadata_api#247` deferred a separate band↔member ("membership") feature.

Two things brought this back open (`lucos_arachne#597`):

1. **The federation half of `#246` was never built.** Traced in code (2026-06-01):
   - **media side:** the `artist` table is `(id INTEGER PK, name TEXT UNIQUE)` only — no column to hold a federation link (`lucos_media_metadata_api` `api/migrations/0001_baseline.sql:48-51`), and `ArtistToRdf` emits only `rdf:type mo:MusicArtist` + `skos:prefLabel` per instance, **no `owl:sameAs`** (`rdfgen/rdf.go`). Artist instance URIs are minted as `{MEDIA_METADATA_MANAGER_ORIGIN}/artists/{id}` (i.e. `media-metadata.l42.eu/artists/<id>`).
   - **arachne side:** the only cross-source reconciliation is the Person-merge built in `lucos_arachne#539`, and it is **`foaf:Person`-scoped on both ends** of every `owl:sameAs` (`ingestor/searchindex.py`, `compute_person_closures`). A `mo:MusicArtist` is a `foaf:Agent`, not a `foaf:Person`, so an artist↔person link would be ignored even if one were emitted.
   - **Net effect:** a human who is both a recording artist and an `eolas:Person` (SRE confirmed David Bowie and Paul McCartney live in the running index, 2026-05-31) appears as **two unlinked nodes under different URIs and types**, with their relationships split across them (e.g. Bowie's `foaf:made` edges on the artist node, `mo:produced` edges on the person node). This is the `#246` design's intended *intermediate* state with the joining link missing — not a new decision, and explicitly **not** a re-opening of the `#246` person-vs-group question.

2. **lucas42 asked that `#597` and `lucos_media_metadata_api#247` be modelled together**, even though implementation may happen separately — hence this single ADR covering both identity and membership.

### The arachne merge mechanism (relevant to the canonical-URI question)

`lucos_arachne`'s Person-merge performs two distinct jobs, and the distinction is load-bearing here:

- **`owl:sameAs` forms the closure.** The connected-component walk in `compute_person_closures` builds its adjacency **only** from `owl:sameAs` pairs — this decides *which URIs collapse into one search document*.
- **`eolas:preferredIdentifier` picks the canonical end.** `_find_primary_uri()` walks `preferredIdentifier` edges to the terminal URI (the one with no outgoing edge) and uses that as the merged document's `id`. `eolas:preferredIdentifier` is declared in `lucos_eolas` (`metadata/views.py`) as an `owl:AsymmetricProperty` whose documented purpose is exactly this, with **domain and range deliberately unconstrained** ("can apply to any URI in the estate") — so it is already legal from a `mo:MusicArtist` URI with no eolas ontology change.
- The merged document stores `secondary_uris`; the explorer item page resolves a request for **any** closure URI via `id:=X || secondary_uris:=X` (`lucos_arachne#567`). So one closure-level change serves **both** the search-index and item-page consumers.

A bare `owl:sameAs` (closure only) with no `preferredIdentifier` between the artist and person URIs would leave `_find_primary_uri` with no edge to walk, falling back to **lexicographic minimum** — a canonical choice made by alphabetical accident rather than principle.

## Decision

The media `Artist` (`mo:MusicArtist`) carries **two semantically distinct kinds of manually-curated, entity-level outbound foreign-URI reference**, both targeting `eolas:Person` URIs and sharing the same storage and reconcile machinery:

### 1. Identity link — 0..1 (`lucos_arachne#597`)

For a **solo artist who is also a person**, media emits, on that `mo:MusicArtist` URI:

- `owl:sameAs` → the `eolas:Person` URI (joins the arachne closure — the predicate `#246` chose), **and**
- `eolas:preferredIdentifier` → the same `eolas:Person` URI (declares the eolas Person the canonical identity, so the merged document's `id` is deterministic, not lexicographic).

This is the federation half of `#246`, now finished. It is **manually curated** — no auto-detection, no ingest-time classification. A group simply never receives an identity link, so **the individual-vs-group distinction is an emergent property of curation, never a write-time decision** (preserving `#237`/`#246`).

### 2. Membership link — 0..n (`lucos_media_metadata_api#247`)

For a **band/group**, media emits `mo:member` from the `mo:MusicArtist` URI → each member's `eolas:Person` URI. These are edges **between** distinct nodes; they cause no merge. Manually curated.

### Shared constraints (both relations)

- Targets are **`eolas:Person` URIs only**, never `contacts:Person` (per `lucos` ADR-0005; the eolas Person carries its own `preferredIdentifier`→contacts link where applicable).
- URIs are host-validated to `eolas.l42.eu` (per `lucos_media_metadata_api#245`).
- Display names are denormalised and refreshed via the existing reconcile mechanism. Per `#247`'s analysis, reconcile currently assumes *tag-level* foreign URIs; both relations need it generalised once to *entity-level* foreign URIs on the Artist.

### arachne generalisation

`lucos_arachne` widens the `#539` closure so a `mo:MusicArtist` / `foaf:Agent` URI can be a closure member and the `preferredIdentifier` canonical-pick is honoured across the `foaf:Agent`↔`foaf:Person` boundary. It must tolerate a group that happens to be typed `foaf:Person` in eolas (SRE flagged U2 modelled that way). Because the merged document carries `secondary_uris`, both the search index and the item page are served by this one change.

### Resulting canonical URI in the search index

- **Individual who is also a person:** the **`eolas:Person` URI** (or the `contacts:Person` URI, if that eolas Person itself has a `preferredIdentifier`→contacts chain). The media `media-metadata.l42.eu/artists/<id>` URI becomes a *secondary*.
- **Group with no person identity:** unchanged — the media `media-metadata.l42.eu/artists/<id>` URI.

### Collaboration "artists"

Composite names like "Paul McCartney and Wings" or "Queen & David Bowie" are **not** one person and must **never** receive an identity (`owl:sameAs`) link — that would wrongly collapse them. They are modelled, where useful, via membership (`mo:member`) edges. The two-relation split is what keeps these out of the identity merge.

## Consequences

### Positive

- **Removes the dual-identity divergence at the root for individuals** — one merged node carrying all of the human's edges, rather than two unlinked nodes with `foaf:made`/`mo:produced` split across them.
- **Reuses mechanisms already in production** — `eolas:preferredIdentifier`, the `#539` Person-merge, and reconcile. `lucos_eolas` needs **no change** (unconstrained `preferredIdentifier` domain). The only genuinely new code is the arachne closure generalisation and the media-side emission/storage.
- **No ingest-time person-vs-group classification** — preserves the `#237`/`#246` decision; `lucos_media_import` is untouched.
- **Identity and membership are designed coherently** against the same eolas-Person anchor. Worked example (Eithne Brennan): "Enya" (solo) carries an *identity* link to the eolas Person; "Clannad" (band) carries a *membership* edge to the same eolas Person — so "who is Enya?" and "what is Eithne a member of?" both resolve against one identity.
- **Principled canonical URI** — eolas as the identity anchor for individuals, by `preferredIdentifier`, not by lexicographic accident.

### Negative

- **Curation burden.** Identity and membership links are hand-populated. An un-curated dual-role human stays divergent until linked. Accepted: this matches `#246`'s deliberate no-auto-detection stance; the alternative (heuristic name-matching) was rejected as fragile.
- **Permanent dual identity for individuals.** A solo artist-who-is-a-person retains two URIs (media + eolas), kept coherent by the `sameAs`/`preferredIdentifier` link. This is inherent to `#246`'s mint-from-name design; the link is what makes it coherent rather than divergent.
- **Over-merge risk in the generalised closure.** Widening past `foaf:Person` must not collapse collaboration artists or name-collisions. Bounded by the fact that the merge keys on **explicit `owl:sameAs` only, never names** — heuristic matching remains rejected.

### Follow-up actions

- **`lucos_arachne#597`** — implementation of the identity link: media_api `owl:sameAs` + `eolas:preferredIdentifier` emission and the supporting schema, plus the arachne `#539` closure generalisation. The work spans **two repos** (`lucos_media_metadata_api` for emission/storage, `lucos_arachne` for the merge); the coordinator may wish to split a `lucos_media_metadata_api` implementation ticket off `#597` when dispatching. Dispatched separately from this ADR.
- **`lucos_media_metadata_api#247`** — implementation of `mo:member` membership modelling. Remains deferred; sits downstream of the identity link existing.
- **Verification (SRE R3, folded into `#597`):** a build-time invariant in the ingestor — a known dual-role human (e.g. David Bowie) yields **exactly one** merged document carrying *both* the `foaf:made` and `mo:produced` edges. Preferred over a name-keyed canary fixture (no-flaky-tests posture).
- The `/v2/export` endpoint-naming hygiene (the lone surviving `/v2/` path) is downstream of the export-contract work and tracked separately, per `#597`.
