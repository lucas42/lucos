# Project Field Reference

Issue metadata for the lucos workflow is managed via custom fields on the [**lucOS Issue Prioritisation** project board](https://github.com/users/lucas42/projects/8), not via GitHub labels. The project board is the source of truth for workflow state, ownership, and prioritisation across all lucos repositories.

The migration from per-repo workflow labels to central project fields was completed in May 2026 — see [lucas42/lucos_claude_config#73](https://github.com/lucas42/lucos_claude_config/issues/73) for the rationale and rollout. Only the `audit-finding` label was retained (see bottom of this page).

For how these fields work together in the issue lifecycle, see [issue-workflow.md](issue-workflow.md).

## Status field

Tracks where an issue sits in its lifecycle.

| Option | Meaning |
|---|---|
| Needs Triage | Default for newly added items. Set automatically when an issue lands on the board. Must be empty at the end of every triage pass. |
| Ideation | Goal or scope is vague/exploratory. Park until revisited. |
| Awaiting Decision | Options have been laid out. Waiting for lucas42 to make a call. |
| Ready | Well-defined, prioritised, and ready for implementation pickup. Also where issues sit while being worked on. |
| Blocked | Well-defined but waiting on another issue. Not available for pickup until the blocker closes. |
| Done | Issue closed. Set automatically by the project board on close. |

## Priority field

Set on **every triaged issue**, not just Ready ones — refinement work also needs prioritising so lucas42 and agents can see which questions are most urgent.

| Option | Meaning |
|---|---|
| Critical | Full service outage — production is down and users are affected right now. Not for important features or bugs with workarounds. |
| High | High impact; should be picked up soon. |
| Medium | Standard priority; normal queue order. |
| Low | Nice to have; pick up when queue is clear. |

No Priority set = not yet prioritised (distinct from Medium). For strategic guidance on which areas of work map to which priority, see [priorities.md](priorities.md).

## Owner field

Indicates who should look at the issue next — used both for refinement routing and for implementation assignment.

| Option | Assigned to |
|---|---|
| lucas42 | Repo owner — product direction, priority calls, decisions only he can make. |
| lucos-architect | Architectural design / review, or ADR / documentation implementation. |
| lucos-system-administrator | Infrastructure and ops. |
| lucos-site-reliability | SRE — monitoring, alerting, reliability, performance, incident management. |
| lucos-security | Cybersecurity. |
| lucos-developer | Implementation — the default for hands-on coding work. |
| lucos-ux | User experience, accessibility, frontend design, copywriting. |
| lucos-code-reviewer | Code review. |
| lucos-issue-manager | Workflow, process documentation, and issue conventions. |

## The one remaining label: `audit-finding`

`audit-finding` is the only workflow-relevant label that still exists. It is applied automatically by the `lucos_repos` audit tool to issues it creates, and indicates the issue tracks a failing repository convention. See [ADR-0002](https://github.com/lucas42/lucos_repos/blob/main/docs/adr/0002-audit-issue-lifecycle.md) for the lifecycle.

It stayed as a label (rather than becoming a project field) because it represents an intrinsic attribute of the issue itself — a failing repo convention — rather than the team's workflow state on top of the issue. That makes it a different shape of metadata, and the project-field model isn't the right home for it.
