# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when assigning `priority:*` labels during triage.

**Last updated**: 2026-04-02

---

## Active priorities (in order)

### 1. lucos_repos and .github (audit & workflow infrastructure)

The `lucos_repos` audit tool and the `.github` shared workflows are the top priority. This includes:

- Improvements to how `lucos_repos` manages audit-finding issues (e.g. lucas42/lucos_repos#248 — auto-closing resolved findings)
- Convention fixes and enhancements in `lucos_repos`
- Shared reusable workflow improvements in `.github` (e.g. tagged releases, naming conventions, SHA pinning, estate rollout gating)
- Issues recently raised by the architect on `.github` (lucas42/.github#34–38)

Issues in `lucos_repos` and `.github` should be treated as `priority:high`.

### 2. lucos_photos and lucos_photos_android

The `lucos_photos` project and its associated repositories (especially `lucos_photos_android`) remain a high priority. The Android app has launched and the service now has production data, making reliability, correctness, and user-facing improvements important.

Issues raised off the back of this launch (e.g. bugs discovered in production use, missing features noticed during real usage, performance issues with real data) should be treated as `priority:high`.

### 3. LLM agent workflows

Tickets that enable or improve LLM agent workflows are the third priority. This includes tooling, infrastructure, and process improvements that make agents more effective at triaging, reviewing, and implementing work.

Examples: agent scripts, monitoring endpoints for agents, agent sandbox provisioning, workflow documentation.

---

## Paused repositories

The following repositories have all non-critical work paused. Issues in these repos should generally be labelled `priority:low` or left unprioritised. Critical security updates are the only exception.

### lucos_auth

All work paused except critical security updates. The service is planned to be replaced entirely with something based around passkeys. New feature work would be wasted effort.

### pici (archived)

Retired. All services migrated to `build-multiplatform` (docker buildx + QEMU). Repo archived. No new issues expected.

---

## How to use this file

Priority labels are assigned to **all issues during triage** -- including `needs-refining` issues, not just `agent-approved` ones. This ensures refinement work is also prioritised.

When assigning priority labels during triage:

- **`priority:critical`**: A production service is completely down and users are affected right now. This is for full service outages only -- not for "very important" features, not for degraded performance, and not for bugs that have workarounds. If you are unsure whether something qualifies, it is not critical. See "Preventing priority inflation" below.
- **`priority:high`**: Issues that fall under priority 1 (lucos_photos, lucos_photos_android, and associated repos), issues raised from the production launch, issues causing a current alert (see "Active alerts" below), or are urgent regardless of area.
- **`priority:medium`**: Issues that fall under priority 2 (agent workflows), or are important but not in the top priority area.
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
