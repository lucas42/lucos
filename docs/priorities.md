# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when setting the Priority field during triage.

**Last updated**: 2026-05-22

---

## Active priorities (in order)

### 1. Media metadata migration to lucos_eolas

A natural extension of the completed `lucos_media` v2 → v3 migration. The next phase moves non-media-specific metadata out of `lucos_media_metadata_api` and into `lucos_eolas` as the canonical home — people (composers, producers, artists), places, content-warning classifications, and other cross-domain entities. The metadata API stays focused on media-specific concerns.

In-flight tickets sit across `lucos_media_metadata_api` (e.g. Person-typed tags, memory, offence, theme_tune/soundtrack predicates, URI rewrite on source change) and `lucos_eolas` (new types as needed). Issues progressing this work should be Priority = High.

### 2. lucos_firewall

Design and rollout of the proposed `lucos_firewall` service, per ADR-0007. Covers the bootstrap of the new repo, the `public_ports` schema and population in `lucos_configy`, RNDC tightening in `lucos_dns`, and the progressive deployment to avalon → xwing → salvare. Issues that progress the firewall design or its rollout should be Priority = High.

---

## Paused repositories

The following repositories have all non-critical work paused. Issues in these repos should generally be set to Priority = Low, or left unprioritised. Critical security updates are the only exception.

### lucos_auth

All work paused except critical security updates. The service is planned to be replaced entirely with something based around passkeys. New feature work would be wasted effort.

---

## How to use this file

The Priority field is set on **every triaged issue** — including refinement-phase issues (Status = Ideation or Awaiting Decision), not just Status = Ready. This ensures refinement work is also prioritised.

When setting Priority during triage:

- **Critical**: A production service is completely down and users are affected right now. This is for full service outages only — not for "very important" features, not for degraded performance, and not for bugs that have workarounds. If you are unsure whether something qualifies, it is not critical. See "Preventing priority inflation" below.
- **High**: Issues that fall under any of the active priorities listed above, issues causing a current alert (see "Active alerts" below), or are urgent regardless of area.
- **Medium**: Important issues not in the top priority areas.
- **Low**: Everything else, including issues in paused repositories.

If an issue in a paused repository has a genuine critical security concern, it may still warrant Priority = High — use judgement.

Within a priority level, oldest issues are picked up first. Ordering can also be overridden by lucas42 via manual board repositioning, which is authoritative.

### Preventing priority inflation

Priority = Critical exists to ensure live outages are always picked up ahead of everything else, no matter how old other issues are. To keep it effective:

- **Do not use it for features**, no matter how important. Use Priority = High instead.
- **Do not use it for bugs that have workarounds.** If users can still accomplish their goal, it is not critical.
- **Do not use it for degraded performance.** Slow is not down.
- **Remove it as soon as the outage is resolved.** Critical issues should not linger — they are either actively being fixed or already closed.

If Priority = Critical is used too broadly, it loses its signal and the queue becomes indistinguishable from Priority = High.

### Active alerts

Any issue that is currently causing an alert — a monitoring alert, a failing health check, a red CI status blocking deploys, or similar — should be triaged at least to Priority = High, regardless of which repository it is in. An active alert overrides the normal priority framework because it represents ongoing impact.

If the alert constitutes a full production outage (service down, users affected), use Priority = Critical as per the existing rules above.

This applies both at initial triage and when an existing issue escalates in impact. If an issue was originally set at Priority = Low or Medium but is now causing an active alert, it should be raised to at least Priority = High.

### Re-assessment after lucas42 input

When lucas42 gives input on an issue (e.g. a comment, a decision, or a reaction), re-assess the Priority. The scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

### Priority override rules

- **lucas42's priority calls override this file.** lucas42 is the repo owner and has final say. If lucas42 explicitly states a priority level for an issue, apply that priority regardless of what this document says.
- **Priority calls from others** (including agents) should be considered within the context of the strategic priorities defined here. They do not automatically override this framework — they are input, not directives.
