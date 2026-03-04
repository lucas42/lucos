# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when assigning `priority:*` labels during triage.

**Last updated**: 2026-03-04

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

### lucos_repos

All work paused. The service is not currently being used for anything. Its purpose needs to be re-thought holistically before adding new features.

---

## How to use this file

When assigning priority labels during triage:

- **`priority:high`**: Issues that fall under priority 1 (agent workflows), or are critical/urgent regardless of area.
- **`priority:medium`**: Issues that fall under priority 2 (lucos_photos and associated repos), or are important but not in the top priority area.
- **`priority:low`**: Everything else, including issues in paused repositories.

If an issue in a paused repository has a genuine critical security concern, it may still warrant `priority:high` -- use judgement.

Within a priority level, oldest issues are picked up first.
