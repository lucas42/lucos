# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when assigning `priority:*` labels during triage.

**Last updated**: 2026-04-06

---

## Active priorities (in order)

### 1. lucos_media_* v3 migration

The migration of `lucos_media_*` services to the v3 API format is the top priority. This includes:

- Completing the v3 migration across `lucos_media_metadata_api`, `lucos_media_metadata_manager`, `lucos_media_manager`, `lucos_media_weightings`, and related services
- Issues tracking dual-format support, consumer migration, and backwards-compatibility removal
- Bug fixes and reliability issues in the media services that arise during or after migration

Issues in `lucos_media_*` repositories related to the v3 migration should be treated as `priority:high`.

### 2. lucos_photos and lucos_photos_android

The `lucos_photos` project and its associated repositories (especially `lucos_photos_android`) remain a high priority. The Android app has launched and the service now has production data, making reliability, correctness, and user-facing improvements important.

Issues raised off the back of this launch (e.g. bugs discovered in production use, missing features noticed during real usage, performance issues with real data) should be treated as `priority:high`.

---

## Paused repositories

The following repositories have all non-critical work paused. Issues in these repos should generally be labelled `priority:low` or left unprioritised. Critical security updates are the only exception.

### lucos_auth

All work paused except critical security updates. The service is planned to be replaced entirely with something based around passkeys. New feature work would be wasted effort.

---

## How to use this file

Priority labels are assigned to **all issues during triage** -- including `needs-refining` issues, not just `agent-approved` ones. This ensures refinement work is also prioritised.

When assigning priority labels during triage:

- **`priority:critical`**: A production service is completely down and users are affected right now. This is for full service outages only -- not for "very important" features, not for degraded performance, and not for bugs that have workarounds. If you are unsure whether something qualifies, it is not critical. See "Preventing priority inflation" below.
- **`priority:high`**: Issues that fall under priority 1 (lucos_media_* v3 migration) or priority 2 (lucos_photos, lucos_photos_android), issues causing a current alert (see "Active alerts" below), or are urgent regardless of area.
- **`priority:medium`**: Important issues not in the top priority areas.
- **`priority:low`**: Everything else, including issues in paused repositories.

If an issue in a paused repository has a genuine critical security concern, it may still warrant `priority:high` -- use judgement.

Within a priority level, oldest issues are picked up first.

### Preventing priority inflation

`priority:critical` exists to ensure live outages are always picked up ahead of everything else, no matter how old other issues are. To keep it effective:

- **Do not use it for features**, no matter how important. Use `priority:high` instead.
- **Do not use it for bugs that have workarounds.** If users can still accomplish their goal, it is not critical.
- **Do not use it for degraded performance.** Slow is not down.
- **Remove it as soon as the outage is resolved.** Critical issues should not linger -- they are either actively being fixed or already closed.

If `priority:critical` is used too broadly, it loses its signal and the queue becomes indistinguishable from `priority:high`.

### Active alerts

Any issue that is currently causing an alert -- a monitoring alert, a failing health check, a red CI status blocking deploys, or similar -- should be triaged at least to `priority:high`, regardless of which repository it is in. An active alert overrides the normal priority framework because it represents ongoing impact.

If the alert constitutes a full production outage (service down, users affected), use `priority:critical` as per the existing rules above.

This applies both at initial triage and when an existing issue escalates in impact. If an issue was originally filed at `priority:low` or `priority:medium` but is now causing an active alert, it should be reprioritised to at least `priority:high`.

### Re-assessment after lucas42 input

When lucas42 gives input on an issue (e.g. a comment, a decision, or a reaction), re-assess the priority. The scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

### Priority override rules

- **lucas42's priority calls override this file.** lucas42 is the repo owner and has final say. If lucas42 explicitly states a priority level for an issue, apply that priority regardless of what this document says.
- **Priority calls from others** (including agents) should be considered within the context of the strategic priorities defined here. They do not automatically override this framework -- they are input, not directives.
