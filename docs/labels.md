# Label Reference

Labels used across lucos repositories to manage the issue workflow. Applied and managed solely by the `lucos-issue-manager` agent.

For how these labels work together in the issue lifecycle, see [issue-workflow.md](issue-workflow.md).

## Workflow labels

| Label | Colour | Purpose |
|---|---|---|
| `agent-approved` | Green (`#0e8a16`) | Issue is clear, well-defined, and ready for implementation. |
| `needs-refining` | Orange (`#d93f0b`) | Issue needs more work before it can be picked up. |

## Automated labels

| Label | Colour | Purpose |
|---|---|---|
| `audit-finding` | Grey (`#ededed`) | Applied automatically by the `lucos_repos` audit tool to issues it creates. Indicates the issue tracks a failing repository convention. See [ADR-0002](https://github.com/lucas42/lucos_repos/blob/main/docs/adr/0002-audit-issue-lifecycle.md) for the lifecycle. |

## Status labels

### Refinement statuses (used with `needs-refining`)

| Label | Colour | Meaning |
|---|---|---|
| `status:ideation` | Light blue (`#c5def5`) | Goal or scope is still vague/exploratory. Park until revisited. |
| `status:needs-design` | Yellow (`#fbca04`) | Goal is clear, but implementation details need fleshing out by an agent. |
| `status:awaiting-decision` | Red (`#b60205`) | Options are laid out. Waiting for lucas42 to make a call. |

### Implementation statuses (used with `agent-approved`)

| Label | Colour | Meaning |
|---|---|---|
| `status:blocked` | Blue (`#1d76db`) | Well-defined but blocked by another issue. Not available for pickup. |

## Owner labels

Indicate who should look at the issue next. Used with both `needs-refining` (review/design) and `agent-approved` (implementation).

| Label | Colour | Assigned to |
|---|---|---|
| `owner:lucas42` | Light olive (`#e4e669`) | Repo owner -- product direction, priority calls, questions only he can answer. |
| `owner:lucos-architect` | Light purple (`#d4c5f9`) | Architectural design/review, or ADR/documentation implementation. |
| `owner:lucos-system-administrator` | Light teal (`#bfdadc`) | Infrastructure/ops. |
| `owner:lucos-site-reliability` | Cream (`#fef2c0`) | SRE -- monitoring, alerting, reliability, performance, incident management. |
| `owner:lucos-security` | Light pink (`#f9d0c4`) | Cybersecurity. |
| `owner:lucos-developer` | Light green (`#c2e0c6`) | Implementation -- the default for hands-on coding work. |
| `owner:lucos-code-reviewer` | | Code review. |
| `owner:lucos-issue-manager` | | Workflow, process documentation, or issue conventions -- the issue manager's review domain. |

## Priority labels

| Label | Colour | Meaning |
|---|---|---|
| `priority:critical` | Dark red (`#e11d48`) | Full service outage -- production is down and users are affected right now. Not for important features or bugs with workarounds. |
| `priority:high` | Red (`#b60205`) | High impact; should be picked up soon. |
| `priority:medium` | Yellow (`#fbca04`) | Standard priority; normal queue order. |
| `priority:low` | Light blue (`#c5def5`) | Nice to have; pick up when queue is clear. |

No priority label = not yet prioritised (distinct from `priority:medium`).

## Repositories

These labels exist in the following repositories:

- lucos
- lucos_agent
- lucos_agent_coding_sandbox
- lucos_backups
- lucos_claude_config
- lucos_configy
- lucos_contacts
- lucos_deploy_orb
- lucos_eolas
- lucos_media_manager
- lucos_media_metadata_manager
- lucos_media_seinn
- lucos_media_weightings
- lucos_monitoring
- lucos_photos
