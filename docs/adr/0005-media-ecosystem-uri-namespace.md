# ADR-0005: Media-ecosystem URI namespace

**Date:** 2026-05-19
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos/issues/163

## Context

The `lucos_media_*` services (notably `lucos_media_metadata_api`, but also `lucos_media_metadata_manager`, `lucos_media_manager`, `lucos_media_weightings`, `lucos_media_import`, and friends) maintain entity references in their persistent stores — most prominently as the `uri` field on v3 tag rows, but also in any future cross-entity relations media services might add (e.g. the planned artist-membership relation in lucas42/lucos_media_metadata_api#247).

These references can in principle point at three kinds of system:

1. **Media-own URIs** — `media:Track/{id}`, `media:Album/{id}`, `media:Artist/{id}` minted by media services themselves.
2. **`lucos_eolas` URIs** — `eolas:Person/{slug}`, `eolas:Place/{slug}`, etc. minted by the encyclopedic-entity service. Tags for `composer`, `producer`, `language`, and similar predicates point at these (see lucas42/lucos_media_metadata_api#237).
3. **`lucos_contacts` URIs** — `contacts:Person/{id}` minted by the contacts service. Used by `lucos_photos` for face linking and by `lucos_calendar` for calendar feeds, but with no settled position on whether they should appear in media-ecosystem stores.

Two recent pieces of work assumed (3) was a legitimate part of the media-ecosystem URI namespace:

- **lucas42/lucos_media_metadata_api#138** added a `contactDeleted` webhook handler on the premise that media tags might hold `contacts.l42.eu` URIs (the "pre-link window" scenario, where a tag was written with a contact URI before the contact was linked to an eolas Person).
- **lucas42/lucos_media_metadata_api#139** (and its draft implementation in PR #244) added a `contactUpdated` handler and a contacts-URI branch in the daily reconcile loop, on the same premise.

The architect chat on 2026-05-19 surfaced two problems with this premise:

1. **The premise is not grounded.** No automated flow actually puts a contacts URI into a media tag. The eolas write API (`eolas:thing_create`) always returns eolas URIs. The media tag-write API does not rewrite URIs. The `lucos_media_metadata_manager` PHP frontend has no contacts integration. The "pre-link window" scenario was hypothesised by the architect when filing #138 and #139, without verification that any code path produced it.
2. **The original framing of the pushback ("contacts is a source of pushes, not pulls") would have conflicted with the `lucos_loganne#370` re-fetch-from-source convention**, which is settled across the estate (`lucos_media_manager#208`, `lucos_media_weightings#148`, `lucos_photos#289` all closed under that pattern). A direct "don't pull from contacts" rule would have required reverting that convention.

A narrower framing avoids both problems: don't constrain how Loganne consumers re-fetch (the existing convention stands); constrain instead what URIs the media ecosystem is allowed to store in the first place.

## Decision

**The media ecosystem stores only its own URIs or `lucos_eolas` URIs in any persistent entity references.** It does not store `lucos_contacts` URIs directly.

This applies to:

- The `uri` field on v3 tag rows in `lucos_media_metadata_api`.
- Any future cross-entity relation in media services that holds an entity URI (e.g. `mo:member` URIs on `media:Artist`, per lucas42/lucos_media_metadata_api#247).
- Any other persistent foreign-URI reference in a `lucos_media_*` service that may be added in future.

Where a media reference needs to surface a `contacts:Person` value (e.g. a personal contact who is also a composer or band member), the media side stores the corresponding `eolas:Person` URI. The eolas Person carries a `preferredIdentifier` link to the `contacts:Person`; `lucos_arachne` performs the inference across that link and surfaces the contact's canonical name for display. The contact is reachable from the media-stored URI in two hops — never in zero.

This means:

- The media ecosystem does not hold the `KEY_LUCOS_CONTACTS` credential.
- Loganne `contactUpdated` and `contactDeleted` events are not consumed by media services (other consumers continue to consume them as appropriate).
- Tag-write boundaries in media services validate the host of any incoming URI against an allowlist of permitted source systems. The allowlist contains the media service's own host (e.g. `media-metadata.l42.eu`) and `eolas.l42.eu` only. `contacts.l42.eu` is **not** on the allowlist.
- The `lucos_loganne#370` re-fetch-from-source convention is unaffected. Media services that hold eolas URIs continue to re-fetch from eolas on `itemUpdated`. They simply never need to re-fetch from contacts because they never hold contacts URIs.

This is a **media-ecosystem-specific** rule. Other services with a legitimate contacts-data use case (e.g. `lucos_photos` for face↔contact linking, `lucos_calendar` for calendar feeds) continue to hold contacts URIs as appropriate. The principle constrains what kinds of cross-system references are appropriate in *media* tag/entity stores, not the existence of cross-system references in general.

## Consequences

### Positive

- **Narrower credential surface.** The media ecosystem does not need `KEY_LUCOS_CONTACTS`. One fewer service holding a contacts credential, one fewer cross-system auth path to maintain and audit.
- **No defensive code for non-existent paths.** The `contactDeleted` handler (added by lucas42/lucos_media_metadata_api#138) and the planned `contactUpdated` handler (lucas42/lucos_media_metadata_api#139) are unused once the rule is enforced. Removal makes the codebase honest about its actual dependency graph.
- **Federation responsibility is clear.** Inferring "contact X is the same as eolas Person Y" belongs to `lucos_arachne`, which already does this work. Media services consume the federated view; they do not re-implement the federation themselves.
- **A simple rule for future agents.** "Media stores its own URIs or eolas URIs" is one sentence, easy to apply during design review and easy to enforce at the tag-write boundary. Replaces ad-hoc judgement on whether each new cross-system reference should reach into contacts.

### Negative

- **One extra eolas record per personal contact who is referenced from media.** Where today a hypothetical direct `contacts:Person` reference would have sufficed, the rule requires an `eolas:Person` record with a `preferredIdentifier` link. This is one record per cross-referenced contact, and creation is the same lookup-or-create pattern used by composer/producer in lucas42/lucos_media_metadata_api#237.
- **Two-hop inference for contact-name display.** Media reference → `eolas:Person` → `preferredIdentifier` → `contacts:Person`. Arachne handles this transparently for SPARQL consumers, but a direct API consumer that wants the canonical contact name from a media-stored URI has to traverse the chain. In practice, this is fine: media stores denormalised names alongside URIs (refreshed via the `itemUpdated` mechanism in lucas42/lucos_media_metadata_api#139), and consumers that need authoritative current names already query arachne.
- **A small constraint on Artist `owl:sameAs` and member URIs.** lucas42/lucos_media_metadata_api#246 and lucas42/lucos_media_metadata_api#247 must point at `eolas:Person`, not `contacts:Person`. Bodies have been amended (2026-05-19) to reflect this.
- **Doesn't generalise estate-wide.** The rule is media-specific. Other services have to make their own call. This is the right scope — `lucos_photos` legitimately needs contacts access for face linking, and a blanket rule would block that — but it does mean architects need to be explicit about scope when applying the principle elsewhere.

### Follow-up actions

- Drop the contacts half of lucas42/lucos_media_metadata_api/pull/244 (developer revising per lucas42's pushback).
- Implement lucas42/lucos_media_metadata_api#245 with `eolas.l42.eu` as the sole non-self host on the tag-write allowlist (body amended 2026-05-19).
- Remove the `contactDeleted` handler from `lucos_media_metadata_api`, tracked in lucas42/lucos_media_metadata_api#248. Includes a sibling deregistration ticket in `lucos_loganne`.
- Update lucas42/lucos_media_metadata_api#246 and lucas42/lucos_media_metadata_api#247 bodies to constrain `owl:sameAs` and member URIs to `eolas:Person` (done 2026-05-19).
- Update lucas42/lucos_media_metadata_api#139 body to drop the `contactUpdated` half (done 2026-05-19).
