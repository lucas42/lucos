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
| `owner:lucos-code-reviewer` | | Code review -- routing issues that require the code reviewer's attention. |
| `owner:lucos-issue-manager` | | Issue management -- routing issues back to the issue manager for triage. |

## How they work together

Every `needs-refining` issue should have exactly one **status** label and one **owner** label. Examples:

- `needs-refining` + `status:ideation` + `owner:lucas42` -- vague idea; lucas42 should revisit when relevant.
- `needs-refining` + `status:needs-design` + `owner:lucos-architect` -- clear goal; architect should flesh out the approach first.
- `needs-refining` + `status:awaiting-decision` + `owner:lucas42` -- options are on the table; lucas42 just needs to pick one.

The key principle: only route to `owner:lucas42` when his input is genuinely needed. If preparatory work can be done by an agent first, route it there with `status:needs-design`.

## Label Management Workflow

**lucos-issue-manager is the sole agent responsible for managing labels.** No other agent should add, remove, or change labels on an issue. This is a deliberate design choice to avoid conflicting label states and ensure there is a single, consistent view of each issue's status.

### Owner agents (system-administrator, architect, code-reviewer, security, site-reliability)

When an owner agent picks up an issue assigned to them via an `owner:*` label, their workflow is:

1. Read the full issue -- body, all comments, any linked PRs.
2. Do the work: post a design proposal, raise a sub-issue, write a PR, provide a security assessment, or whatever the issue calls for.
3. Post a summary comment on the issue describing:
   - What was done
   - What the agent believes the next step is (e.g. "I've laid out three options above -- a decision is needed before implementation can begin")
4. **Do not touch any labels.** Leave label management entirely to lucos-issue-manager.

### lucos-issue-manager

On its next triage pass, lucos-issue-manager reviews the issue and transitions labels based on the current state:

- **If the issue is ready to work on**: remove `needs-refining`, the current `status:*` label, and the `owner:*` label; add `agent-approved`.
- **If more work is needed from a different agent**: update the `status:*` and `owner:*` labels to route to the next person.
- **If the work was incomplete or unclear**: leave labels as-is, or add a comment requesting clarification.

#### Detecting completed agent work

When an issue is owned by an agent (e.g. `owner:lucos-architect`) and that agent's comment is the most recent activity on the issue with no subsequent reply, this is a signal that the agent has finished their work. The issue manager should review the comment and transition labels accordingly:

- If the agent's proposal or work is clearly complete and uncontroversial, transition directly to `agent-approved`.
- If the agent has laid out options or a design that needs sign-off from lucas42, transition to `status:awaiting-decision` + `owner:lucas42`.
- If the agent's work is incomplete or raises further questions, leave the labels as-is or add a comment requesting clarification.

The key insight is that the issue manager should not wait for an explicit "I'm done" signal beyond the agent's summary comment -- the comment itself is the signal.

#### Reactions as approval

If lucas42 adds a +1 reaction to a comment, treat that as approval of the recommendations in that comment. This applies to any comment, but is most relevant when an agent has posted a design proposal or set of recommendations:

- If the +1'd comment contains a design proposal or implementation plan, the issue manager should treat the design as approved and transition the issue to `agent-approved` (assuming no other blocking questions remain).
- If the +1'd comment lays out multiple options with a recommendation, treat the +1 as agreement with the recommended option.

This avoids requiring lucas42 to write a full text reply when a simple thumbs-up conveys the same intent. The issue manager should check for reactions on comments when reviewing issues, not just the text of replies.

### When a PR closes the issue

If a PR with a `Closes #N` (or equivalent) keyword is merged, GitHub automatically closes the issue. Labels on closed issues are largely moot -- lucos-issue-manager does not need to triage them.

### Why this separation exists

Having a single agent own all label transitions means there is always a consistent, auditable record of why an issue is in its current state. Owner agents doing their own label management risks conflicting state -- for example, an agent marking something `agent-approved` immediately after doing design work, before anyone has verified the design is sufficient. Routing everything back through lucos-issue-manager provides that checkpoint.

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
