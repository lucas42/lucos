# Label Scheme

Labels used across lucos repositories to manage the issue workflow. Applied by the `lucos-issue-manager` agent during triage and review.

## Workflow labels

| Label | Colour | Purpose |
|---|---|---|
| `agent-approved` | Green (`#0e8a16`) | Issue is clear, well-defined, and ready for implementation. |
| `needs-refining` | Orange (`#d93f0b`) | Issue needs more work before it can be picked up. See status and owner labels below. |

## Status labels

Applied alongside `needs-refining` or `agent-approved` to classify the issue's current state.

### Refinement statuses

Used with `needs-refining` to classify **why** the issue is not yet ready.

| Label | Colour | Meaning | Priority |
|---|---|---|---|
| `status:ideation` | Light blue (`#c5def5`) | Goal or scope is still vague/exploratory. Park until someone revisits with a clearer picture. | Low |
| `status:needs-design` | Yellow (`#fbca04`) | Goal is clear, but implementation details need fleshing out -- typically by an agent before lucas42 needs to weigh in. | Medium |
| `status:awaiting-decision` | Red (`#b60205`) | Thorough discussion has happened and options are laid out. Waiting for lucas42 to make a call. | **Highest for lucas42** |

### Implementation statuses

Used with `agent-approved` to indicate an issue that is well-defined but not yet available for pickup.

| Label | Colour | Meaning |
|---|---|---|
| `status:blocked` | Blue (`#1d76db`) | Well-defined and implementation-ready, but blocked by another issue that must be completed first. The blocking issue should be referenced in the issue body or a comment. |

When a blocking issue is closed, lucos-issue-manager removes `status:blocked` on its next triage pass, making the issue available for pickup.

## Owner labels

Indicate **who should look at the issue next**. Used alongside both `needs-refining` (for review/design work) and `agent-approved` (for implementation work).

When an issue has `needs-refining`, the owner is expected to review, design, or comment. When an issue has `agent-approved`, the owner is expected to implement it -- write code, create a PR, and ship it.

| Label | Colour | Assigned to |
|---|---|---|
| `owner:lucas42` | Light olive (`#e4e669`) | Repo owner -- product direction, priority calls, questions only he can answer. |
| `owner:lucos-architect` | Light purple (`#d4c5f9`) | Architectural design/review -- data modelling, API contracts, cross-service interactions. |
| `owner:lucos-system-administrator` | Light teal (`#bfdadc`) | Infrastructure/ops -- Docker config, deployment, server setup. |
| `owner:lucos-site-reliability` | Cream (`#fef2c0`) | SRE -- monitoring, alerting, reliability, performance. |
| `owner:lucos-security` | Light pink (`#f9d0c4`) | Cybersecurity -- authentication, authorisation, data protection, vulnerabilities. |
| `owner:lucos-developer` | Light green (`#c2e0c6`) | Implementation -- the default persona for hands-on coding work. |
| `owner:lucos-code-reviewer` | | Code review -- routing issues that require the code reviewer's attention. |
| `owner:lucos-issue-manager` | | Issue management -- routing issues back to the issue manager for triage. |

## Priority labels

Indicate relative importance. Applied by lucos-issue-manager during triage, based on issue context and any input from lucas42.

| Label | Colour | Meaning |
|---|---|---|
| `priority:high` | Red (`#b60205`) | High impact on users or other work; should be picked up soon. |
| `priority:medium` | Yellow (`#fbca04`) | Standard priority; pick up in normal queue order. |
| `priority:low` | Light blue (`#c5def5`) | Nice to have; only pick up when the queue is otherwise clear. |

Issues without a priority label have not yet been prioritised. This is distinct from `priority:medium` -- an unprioritised issue should be prioritised before being picked up for implementation.

When picking up work, agents should process issues in priority order: `priority:high` first, then `priority:medium`, then `priority:low`. Within the same priority level, oldest issues first.

## How they work together

### Refinement phase

Every `needs-refining` issue should have exactly one **status** label and one **owner** label. Examples:

- `needs-refining` + `status:ideation` + `owner:lucas42` -- vague idea; lucas42 should revisit when relevant.
- `needs-refining` + `status:needs-design` + `owner:lucos-architect` -- clear goal; architect should flesh out the approach first.
- `needs-refining` + `status:awaiting-decision` + `owner:lucas42` -- options are on the table; lucas42 just needs to pick one.

The key principle: only route to `owner:lucas42` when his input is genuinely needed. If preparatory work can be done by an agent first, route it there with `status:needs-design`.

### Implementation phase

Every `agent-approved` issue should have an **owner** label indicating who will implement it, and optionally a **priority** label. Examples:

- `agent-approved` + `owner:lucos-developer` + `priority:high` -- ready for the developer to pick up as a high-priority item.
- `agent-approved` + `owner:lucos-system-administrator` -- infrastructure-only change assigned to sysadmin.
- `agent-approved` + `owner:lucos-developer` + `status:blocked` -- ready but waiting on a dependency; do not pick up yet.

### Implementation assignment guidelines

When marking an issue `agent-approved`, lucos-issue-manager also assigns the `owner:*` label for implementation. The default is `owner:lucos-developer`. Exceptions:

- **Purely infrastructure changes** (Docker config, deployment, server setup with no application code): `owner:lucos-system-administrator`.
- **Purely monitoring/logging/pipeline work** (deployment pipelines, alerting, observability with no application code): `owner:lucos-site-reliability`.
- **Purely security work** (authentication setup, vulnerability remediation with no application code): `owner:lucos-security`.
- **Mixed work** (infrastructure + coding, security + coding, etc.): `owner:lucos-developer`. Ensure the relevant specialist has reviewed the issue first.
- **If unclear**: `owner:lucos-developer`.

### Specialist follow-up routing

Some issues need review from a specialist agent **after** the primary owner has finished their work, but **before** the issue is marked `agent-approved`. This applies to two domains: observability/reliability (SRE) and security.

#### SRE follow-up on observability issues

When an issue touches **monitoring, logging, observability, or reliability**, it should be routed to the appropriate primary owner as normal (e.g. `owner:lucos-architect` for design, `owner:lucos-system-administrator` for infrastructure). However, after the primary owner's work is complete, the issue is **re-routed to `owner:lucos-site-reliability`** for SRE review before being marked `agent-approved`.

This also applies mid-lifecycle: if observability or reliability concerns are raised in a comment after the initial triage (e.g. an architect proposes a design and a reliability concern is flagged in follow-up), the issue is routed to SRE for review regardless of the original topic.

The goal is to ensure SRE always gets to weigh in on issues that affect monitoring, logging, or system reliability -- without displacing the primary owner who does the initial work.

#### Security follow-up on security-sensitive issues

When an issue touches **authentication, authorisation, data protection, secret management, or other security topics**, it should be routed to the appropriate primary owner as normal. However, after the primary owner's work is complete, the issue is **re-routed to `owner:lucos-security`** for security review before being marked `agent-approved`.

This also applies mid-lifecycle: if security concerns are raised in a comment after the initial triage (e.g. an architect proposes a design and someone raises an authentication or data protection concern in follow-up), the issue is routed to security for review regardless of the original topic.

The goal is to ensure the security agent always gets to weigh in on issues that affect authentication, authorisation, data handling, or other security-sensitive areas -- without displacing the primary owner who does the initial work.

## Label Management Workflow

**lucos-issue-manager is the sole agent responsible for managing labels.** No other agent should add, remove, or change labels on an issue. This is a deliberate design choice to avoid conflicting label states and ensure there is a single, consistent view of each issue's status.

### Review agents (architect, system-administrator, code-reviewer, security, site-reliability)

When a review agent picks up an issue assigned to them via an `owner:*` label (with `needs-refining`), their workflow is:

1. Read the full issue -- body, all comments, any linked PRs.
2. Do the work: post a design proposal, raise a sub-issue, write a PR, provide a security assessment, or whatever the issue calls for.
3. Post a summary comment on the issue describing:
   - What was done
   - What the agent believes the next step is (e.g. "I've laid out three options above -- a decision is needed before implementation can begin")
4. **Do not touch any labels.** Leave label management entirely to lucos-issue-manager.

### Implementation agents (developer, system-administrator, site-reliability, security)

When an implementation agent picks up an issue assigned to them via an `owner:*` label (with `agent-approved`), their workflow is:

1. Post a starting comment explaining their approach.
2. Create a branch, implement the changes, write tests, and open a PR with a closing keyword (e.g. `Closes #42`).
3. The PR merge automatically closes the issue -- no manual label changes needed.

Note: some personas (system-administrator, site-reliability, security) can act as both reviewers and implementers depending on the issue's workflow labels.

### lucos-issue-manager

On its next triage pass, lucos-issue-manager reviews the issue and transitions labels based on the current state:

- **If the issue is ready to work on**: remove `needs-refining`, the current `status:*` label, and the review-phase `owner:*` label; add `agent-approved` and the implementation `owner:*` label.
- **If more work is needed from a different agent**: update the `status:*` and `owner:*` labels to route to the next person.
- **If the work was incomplete or unclear**: leave labels as-is, or add a comment requesting clarification.
- **If a `status:blocked` issue's dependency has been resolved**: remove `status:blocked` to make it available for pickup.

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
- pici
