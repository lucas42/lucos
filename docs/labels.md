# Label Scheme

Labels used across lucos repositories to manage the issue workflow. Applied by the `lucos-issue-manager` agent during triage and review.

## Workflow labels

| Label | Colour | Purpose |
|---|---|---|
| `agent-approved` | Green (`#0e8a16`) | Issue is clear, well-defined, and ready for implementation. |
| `needs-refining` | Orange (`#d93f0b`) | Issue needs more work before it can be picked up. See status and owner labels below. |

## Status labels

Applied alongside `needs-refining` to classify **why** the issue is blocked.

| Label | Colour | Meaning | Priority |
|---|---|---|---|
| `status:ideation` | Light blue (`#c5def5`) | Goal or scope is still vague/exploratory. Park until someone revisits with a clearer picture. | Low |
| `status:needs-design` | Yellow (`#fbca04`) | Goal is clear, but implementation details need fleshing out -- typically by an agent before lucas42 needs to weigh in. | Medium |
| `status:awaiting-decision` | Red (`#b60205`) | Thorough discussion has happened and options are laid out. Waiting for lucas42 to make a call. | **Highest for lucas42** |

## Owner labels

Applied alongside `needs-refining` to indicate **who should look at the issue next**.

| Label | Colour | Assigned to |
|---|---|---|
| `owner:lucas42` | Light olive (`#e4e669`) | Repo owner -- product direction, priority calls, questions only he can answer. |
| `owner:lucos-architect` | Light purple (`#d4c5f9`) | Architectural design/review -- data modelling, API contracts, cross-service interactions. |
| `owner:lucos-system-administrator` | Light teal (`#bfdadc`) | Infrastructure/ops -- Docker config, deployment, server setup. |
| `owner:lucos-site-reliability` | Cream (`#fef2c0`) | SRE -- monitoring, alerting, reliability, performance. |
| `owner:lucos-security` | Light pink (`#f9d0c4`) | Cybersecurity -- authentication, authorisation, data protection, vulnerabilities. |

## How they work together

Every `needs-refining` issue should have exactly one **status** label and one **owner** label. Examples:

- `needs-refining` + `status:ideation` + `owner:lucas42` -- vague idea; lucas42 should revisit when relevant.
- `needs-refining` + `status:needs-design` + `owner:lucos-architect` -- clear goal; architect should flesh out the approach first.
- `needs-refining` + `status:awaiting-decision` + `owner:lucas42` -- options are on the table; lucas42 just needs to pick one.

The key principle: only route to `owner:lucas42` when his input is genuinely needed. If preparatory work can be done by an agent first, route it there with `status:needs-design`.

## Repositories

These labels exist in the following repositories:

- lucos
- lucos_backups
- lucos_contacts
- lucos_eolas
- lucos_media_manager
- lucos_media_seinn
- lucos_media_weightings
- lucos_photos
- pici
