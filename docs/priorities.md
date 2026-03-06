# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when assigning `priority:*` labels during triage.

**Last updated**: 2026-03-05

---

## Active priorities (in order)

### 1. LLM agent workflows

Tickets that enable or improve LLM agent workflows are the highest priority. This includes tooling, infrastructure, and process improvements that make agents more effective at triaging, reviewing, and implementing work.

Examples: agent scripts, monitoring endpoints for agents, agent sandbox provisioning, workflow documentation.

### 2. lucos_photos

The `lucos_photos` project and its associated repositories (e.g. `lucos_photos_android`) are the next priority. This is the primary user-facing project under active development.

---

## Paused repositories

The following repositories have all non-critical work paused. Issues in these repos should generally be labelled `priority:low` or left unprioritised. Critical security updates are the only exception.

### lucos_auth

All work paused except critical security updates. The service is planned to be replaced entirely with something based around passkeys. New feature work would be wasted effort.

### pici

All work paused. Being retired in favour of docker buildx multi-platform builds (see [lucos_deploy_orb#9](https://github.com/lucas42/lucos_deploy_orb/issues/9)).

---

## How to use this file

Priority labels are assigned to **all issues during triage** -- including `needs-refining` issues, not just `agent-approved` ones. This ensures refinement work is also prioritised.

When assigning priority labels during triage:

- **`priority:critical`**: A production service is completely down and users are affected right now. This is for full service outages only -- not for "very important" features, not for degraded performance, and not for bugs that have workarounds. If you are unsure whether something qualifies, it is not critical. See "Preventing priority inflation" below.
- **`priority:high`**: Issues that fall under priority 1 (agent workflows), or are urgent regardless of area.
- **`priority:medium`**: Issues that fall under priority 2 (lucos_photos and associated repos), or are important but not in the top priority area.
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

### Re-assessment after lucas42 input

When lucas42 gives input on an issue (e.g. a comment, a decision, or a reaction), re-assess the priority. The scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

### Priority override rules

- **lucas42's priority calls override this file.** lucas42 is the repo owner and has final say. If lucas42 explicitly states a priority level for an issue, apply that priority regardless of what this document says.
- **Priority calls from others** (including agents) should be considered within the context of the strategic priorities defined here. They do not automatically override this framework -- they are input, not directives.
