# ADR-0013: LucOS File Uploader — stateless uploader with a uniform backend ingest contract

**Date:** 2026-06-08
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos/issues/209

## Context

There is no single place to get a file *into* the estate. Photos arrive via the Android backup app hitting `lucos_photos`' `POST /photos`. Music arrives as files dropped onto the NAS, where `lucos_media_import` scans them and writes metadata to `lucos_media_metadata_api`. Non-public documents are served by `lucos_private` from the NAS. Each ingest path was built independently, in isolation, for one file type.

The request (lucas42/lucos#209) is for a **one-stop file uploader**: a single web UI where you drag files (or archives — a Bandcamp album zip, say), tag them with a little metadata, and have each routed to the right home. The headline constraints from the ticket are:

- **Stateless** — the uploader holds no durable record of what has been uploaded; it is not a new system-of-record.
- **Pluggable / extensible for new upload types** — adding a new kind of upload should not require rewriting the uploader.
- **Auto-decompression / extraction** — archives are expanded and their constituents routed individually.
- **Drag-and-drop with "folders"** as discrete-metadata selectors, plus per-file metadata.
- **Batch uploads** at non-trivial scale.

The danger with a "one-stop" anything is that it becomes a coupling hub — a single component that accretes knowledge of every backend's API, every file type's quirks, and every storage location's permissions, until changing any backend means changing the uploader. The estate's existing grain is the opposite: **each system owns its own domain** (its storage, its metadata schema, its validation, its permissions). The whole design problem is to build the uploader *without* violating that grain.

This ADR settles the architecture. lucas42 answered six product / one-way-door questions on the ticket (2026-06-08); their resolutions are folded in below. The build decomposition that follows from this ADR is the next phase (see Follow-up actions) — this document is the frame that phase sits inside.

## Decision

### Three tiers of responsibility

The word doing the work in the spec is **stateless**. "Stateless" means the uploader holds no *durable* state — no database, no record of what's been uploaded, no ownership of files. It does need *transient* working space mid-operation (bytes in flight, an extracted archive's contents); that is a bounded resource concern, not durable state. Drawing that line gives three tiers:

1. **Browser — the stateful coordinator.** Holds the upload *session*: the file list, the metadata assigned to each file, per-file progress, and which files failed. It drives the uploads and retries failures. The session state lives here, not in the uploader. This is what makes the uploader's statelessness honest: there is no half-written uploader state to reconcile after a crash — the browser holds the truth and re-drives.
2. **Uploader service — stateless transform + route.** Receives a file (or archive) + a declared upload type + metadata; extracts archives server-side if needed; routes each resulting file + its metadata to the correct backend's ingest endpoint; returns a per-file result. No persistence; bounded transient working space, cleaned up after each operation.
3. **Backends — durable store + metadata validation + permissions.** Own the bytes, own their metadata schema, own where files land on disk, own duplicate detection. Each exposes an **ingest endpoint** conforming to the uniform contract below.

The discipline this protects: the uploader stays **thin**. The moment it grows backend-specific code it becomes the coupling hub we are trying to avoid.

### The uniform ingest contract (the central decision)

Backends are plugged in via a **uniform ingest contract**, not via adapter plugins inside the uploader. Concretely:

- Each backend exposes a **standard ingest endpoint** that accepts a file plus a per-file metadata blob, and returns a per-file result (accepted / rejected / duplicate, with a reason).
- Each backend **declares its own metadata schema** — field names, field kinds, and for discrete fields the allowed values — and **advertises that schema from a discovery endpoint the backend itself owns.** The schema is owned by, validated by, and served by the backend. It is the single source of truth.
- The uploader holds only a **thin registry of backend ingest endpoints** (which backends exist and their URLs). It fetches each backend's metadata schema from the backend at render time and is otherwise fully generic.

Two consequences fall straight out of this and are the reason for the choice:

- **The UI "folders" become backend-driven.** The music backend declares `provenance: discrete [bandcamp, 7digital, qobuz]`; the uploader renders those as folders automatically. Adding a provenance value (`qobuz`) is a backend-side change the uploader picks up on its next schema fetch — it never touches the uploader or its config.
- **"Extensible" means configuration, not code.** Adding a whole new upload *type* = a backend implements the ingest contract, declares its schema, and is registered in the uploader's thin URL registry (one entry). No uploader code change, no redeploy of uploader logic to add a field or a value.

This matches the estate's one-system-owns-its-domain grain exactly: the backend owns its ingest, its schema, and its validation; the uploader owns routing and nothing else.

### Where the schema config lives (lucas42 left this to us)

lucas42 was "not fussy" and named three contenders: YAML in the uploader repo, `lucos_configy`, or `/_info`-style endpoints. The decision is **backend-owned and backend-advertised**, on a dedicated discovery endpoint:

- **Not YAML in the uploader repo** — that re-introduces the coupling Approach A exists to avoid: the uploader repo becomes the registry of every backend's schema, and a backend schema change requires an uploader-repo change one repo away from where the validation actually lives. Two sources of truth that can drift.
- **Not `lucos_configy`** — configy is for host / domain / deployment topology, not per-service functional API schemas. Wrong grain; it would make configy a dependency on the hot path of rendering the upload form.
- **A dedicated discovery endpoint, not `/_info`** — this is closest to lucas42's "`/_info` endpoints" contender, and the instinct (the backend advertises) is right; but `/_info` is the estate's *monitoring* contract with a defined tiered schema, and overloading it with a functional ingest schema mixes two concerns that evolve independently. A dedicated `GET`-able schema endpoint on the ingest surface keeps monitoring and functional contract separate.

The thin **registry of backend URLs** (small, changes only when a whole backend is added) is fine as config in the uploader repo or configy — it is estate topology, not per-service schema, so it does not carry the drift problem.

### Server-side extraction

Archive extraction happens **server-side, in the uploader** (lucas42: prefer server side). The uploader expands an archive and forwards the *constituent files* — each with relative-path metadata so album/folder structure is preserved — to the backend ingest endpoint. This is more robust than client-side JS, handles the folder-structure and permissions story cleanly (see below), and keeps the browser's job to coordination rather than decompression.

### Metadata is two-tiered

The contract carries **per-file** metadata, not just a per-batch blob, because the spec mixes two kinds:

- **Discrete, per-folder** — music provenance, document category — chosen by dragging files into a folder (the backend-declared discrete fields).
- **Free, per-file** — a photo's approximate date-taken, a note on the reverse of a scan — edited per file.

The detailed interaction design (how the folders and the per-file editor compose) is UX's call; the architectural requirement is only that the contract is per-file.

### Definition of "done" (Q1) — hand-off, not end-to-end ingest

A successful upload means **"handed off successfully"**, not "fully ingested." For a synchronous backend (`lucos_photos`) those coincide. For an asynchronous backend (music lands on the NAS and is scanned *later* by `lucos_media_import`), "success" means the file landed at its destination, not that it appears in the collection. Each backend logs its own domain's notion of "fully ingested" to `lucos_loganne`, so the user can verify completion there. The async ingest endpoints therefore follow ADR-0006 (accept-202-enqueue): accept the file fast (it has landed), let the slower scan / metadata step run behind the receive boundary and announce completion via loganne.

### Attribution (Q2) — a logging concern, not a stored column

lucas42 clarified: who uploaded a file should be **determinable from logs** (which user was logged in, which system it went through), but it is **not stored in the database**. So deposited records are *not* tagged with an uploader-identity / contact-ID provenance column in any backend's data model. The uploader logs the authenticated user and the destination backend; backends log their ingest events. This keeps the data model clean and puts provenance where lucas42 wants it — in the audit trail, not the schema.

### Batch failure model (Q3) — partial success + retry

A batch is **partial-success**, not all-or-nothing. For a 500-file batch where 3 fail, the 497 succeed and the browser (the stateful coordinator) retries the 3 failures. The uploader returns a per-file result; the browser owns the retry loop. No batch-level transaction, no rollback of the 497.

### Duplicate handling (Q6) — the backend's job

Duplicate detection is **entirely the backend's responsibility**. The uploader does not dedup; it surfaces a backend's "duplicate" response gracefully (the estate already has the 409-on-duplicate precedent in `lucos_eolas`). What "duplicate" means is the backend's call — `lucos_photos` dedups on exact SHA-256 today; another backend may use a different notion. The uploader only needs the contract to carry a distinct "duplicate" result so the browser can show it without treating it as an error to retry.

### Authentication (the #132 tie-in)

The uploader is **both** an auth roles at once: a human-auth *consumer* (you log in to use it) and a *machine principal* to the backends (it deposits files on your behalf, server-side). Both map onto the session model being designed in lucas42/lucos#132. The uploader can ship behind current auth and migrate to #132 later, but the backend-ingest-auth half is a genuine future #132 dependency, not a coupling to resolve now. Per the #132 direction, the uploader→backend call presents a *signed, scoped session*; the ingest scope is exactly the narrow capability such a machine principal should hold.

## Scope

This ADR decides the **architecture**: the three-tier model, the uniform ingest contract, schema ownership, server-side extraction, and the resolutions of the six product questions. It is the frame for the build.

It does **not** specify: the precise wire format of the contract (field-type vocabulary, the exact shape of the schema-discovery response, the per-file result object); the uploader's UI interaction design; or the per-backend NAS layout. Those are detailed-design decisions for the build tickets, which conform to this frame.

## Alternatives considered

### Approach B — adapter plugins inside the uploader

Give the uploader per-backend code that knows each backend's existing API directly. More immediately flexible for heterogeneous backends — no backend needs a new endpoint, the uploader just speaks each backend's current language.

**Rejected.** Every new upload type becomes a code change plus a redeploy of the uploader, and the uploader slowly accretes knowledge of every backend until it is the coupling hub the whole design is trying to avoid. It also inverts the estate's grain: backend-specific validation and schema would live in the uploader rather than in the backend that owns the domain. The uniform contract costs more up front (each filesystem backend needs a thin ingest endpoint) but keeps the uploader generic forever.

### Schema declared in the uploader repo (YAML)

Hold each backend's metadata schema as YAML in the uploader's own repo.

**Rejected.** Convenient to start, but it is two sources of truth for one fact: the uploader's YAML and the backend's actual validation logic, which drift the moment a backend changes a field. It also re-couples — adding a provenance value means editing the uploader repo, exactly the code-change-to-extend outcome Approach A exists to prevent. Schema belongs with the system that validates against it.

### Schema in `lucos_configy`

Centralise the schemas in the estate config service.

**Rejected.** configy's domain is deployment topology (hosts, domains, volumes), not per-service functional API contracts. Putting ingest schemas there gives the wrong grain and adds configy to the hot path of rendering the upload form. The backend already owns and validates the schema; it should serve it.

### Client-side extraction

Extract archives in browser JS, uploading the constituents directly.

**Rejected** (lucas42 prefers server-side). Less robust, pushes decompression and the folder-structure/permissions handling into the least controlled environment, and complicates the browser's role, which should stay coordination.

### Storing uploader-identity in the backend data model

Tag each deposited record with the uploading user's contact ID for provenance.

**Rejected** (lucas42's Q2 answer). Provenance is wanted in logs, not the schema. Adding a contact-ID column to every backend's data model for this would be schema bloat for a fact the audit trail already carries.

## Consequences

### Positive

- **The uploader stays thin and generic, permanently.** It routes and extracts; it never learns a backend's schema or API. New upload types and new metadata values are configuration / backend-side changes, never uploader code.
- **The estate's grain is preserved.** Each backend keeps ownership of its storage, schema, validation, permissions, and duplicate policy. The uploader does not erode the one-system-owns-its-domain boundary; it sits on top of it.
- **Clean reliability story.** Statelessness means no uploader state to reconcile after a crash; the browser re-drives. Partial-success + browser retry means one bad file never fails a 497-file batch.
- **The Bandcamp 700-permission problem likely disappears by construction.** Because the uploader extracts the archive and forwards constituent files (with relative paths) to the ingest endpoint, destination directories are created *fresh by the ingest endpoint under its own user/umask* — not copied with a restrictive mode from the source archive. The explicit `chmod 770` becomes a belt-and-braces line in one well-defined place (the NAS-write boundary, inside the ingest endpoint) rather than a manual post-upload step, and the uploader never needs to know the serving user/group.
- **Async backends report honestly.** "Handed off" + loganne-confirms-ingest (ADR-0006 shape) means the user gets a fast, truthful acknowledgement and a real completion signal, instead of the uploader pretending a not-yet-scanned music file is "in the collection."

### Negative

- **Each filesystem-backed backend needs a new ingest endpoint it does not have today.** This is the bulk of the real work, and it is honest cost. In particular `lucos_media_import` is currently a *scanner with no HTTP surface at all* — giving it (or a sibling component) an ingest endpoint means introducing an HTTP server to a system that is presently a batch job. That is a non-trivial architectural addition, not a small handler. `lucos_private` already serves over HTTP, so a documents ingest endpoint there is a smaller step; `lucos_photos` already has `POST /photos` and needs *conforming* to the contract rather than building from scratch.
- **The schema-discovery round-trip is a new runtime dependency.** Rendering the upload form requires fetching each backend's schema. If a backend is down, its folders can't render. This is acceptable (you can't upload to a down backend anyway) but the uploader must degrade gracefully — show the backends that *are* reachable, not fail the whole form.
- **Server-side extraction is a real attack surface, and it lives in the uploader.** Ingesting arbitrary archives means **zip-slip / path traversal** (extracted paths must be sanitised — no escaping the working dir, no symlink escape) and **zip-bombs** (bounded expansion: size and file-count ceilings, streamed extraction, a hard temp-space budget with cleanup). These are must-design-in, not later hardening. The uploader is the one component handling untrusted bytes in bulk, so this surface is concentrated there by design — which is at least better than spreading it across every backend.
- **The uploader is a machine principal to every backend.** It can deposit files into multiple personal data stores on the user's behalf. That makes its credential (and, under #132, its scoped session) a meaningful blast radius — narrower than a human's, but real. Its ingest scope must be exactly that and nothing more.
- **No single backend can confirm end-to-end completion synchronously for async types.** The user must consult loganne to know a music upload is fully ingested. This is the correct trade-off (the uploader cannot wait on a scan that may run minutes later) but it does mean "the file is uploaded" and "the track is in my collection" are two distinct, separately-observable events.

### Out of scope

- **The precise contract wire format** — field-type vocabulary, schema-discovery response shape, per-file result object. Detailed design for the uploader build ticket, which is the contract's reference implementation; the backend tickets conform to it.
- **The uploader's UI interaction design** — how folders and the per-file metadata editor compose. UX's call, on the build ticket.
- **The per-backend NAS layout** — exactly where music vs documents land and which component owns each write path. To be confirmed with lucas42 / sysadmin during the backend build tickets; this ADR fixes only the *principle* (the perm fix lives at the NAS-write boundary, inside the ingest endpoint, under its own umask).
- **The #132 auth migration itself** — tracked on lucas42/lucos#132. The uploader can ship behind current auth and adopt the machine-principal session when #132 lands.

### Follow-up actions

The build decomposition is the next phase, and — following the ADR-0006 precedent of not raising tickets ahead of the decision they depend on — these are **raised once this ADR is accepted (the draft PR is signed off and merged)**, not before, to avoid churn if review reshapes the design. They are enumerated here so nothing is lost; the coordinator owns their creation and board placement on acceptance:

1. **Create `lucos_file_uploader`** (new system, raised on `lucas42/lucos`): the stateless uploader service + the browser coordinator. Defines the contract wire format as its reference implementation. Pulls in UX for the folders / per-file-metadata interaction. The server-side-extraction security surface (zip-slip, zip-bomb) is in scope here and should get a security review.
2. **`lucos_photos`: conform `POST /photos` to the ingest contract** — advertise its metadata schema on a discovery endpoint and accept the uniform per-file metadata blob. Smallest of the backend changes (the endpoint already exists).
3. **`lucos_media_import`: music ingest endpoint** — the larger one, because it means giving a currently-HTTP-less scanner an ingest surface (or a sibling component that writes to the scan location). Carries the Bandcamp `chmod 770`-at-the-NAS-write-boundary fix. Pull in sysadmin on the NAS write path.
4. **`lucos_private`: documents ingest endpoint** — already serves over HTTP, so the smaller of the two filesystem backends.

Items 1–4 are **blocked on this ADR being accepted**, and items 2–4 are downstream of item 1 defining the contract. When the ADR merges I will raise them and hand the URLs to the coordinator for triage.

## References

- lucas42/lucos#209 — the originating request and the six product/one-way-door answers (lucas42, 2026-06-08).
- ADR-0006 (Webhook consumer accept-202-enqueue) — the pattern the asynchronous ingest endpoints follow for "handed off, confirmed later via loganne."
- lucas42/lucos#132 — the passkey auth service; source of the machine-principal session the uploader uses for backend ingest.
- lucas42/lucos_eolas — the 409-on-duplicate precedent the backends' duplicate handling follows.
- `lucas42/lucos_photos` `POST /photos` (`api/app/routers/photos.py`) — the existing photo ingest path, to be conformed to the contract.
- `lucas42/lucos_media_import` — the filesystem audio scanner that today has no HTTP ingest surface.
- `lucas42/lucos_private` — serves non-public NAS files over HTTP; natural home for the documents ingest endpoint.
