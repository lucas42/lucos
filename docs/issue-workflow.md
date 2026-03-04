# Issue Workflow

How issues move through the lucos pipeline from creation to completion.

The persona instruction files (under `~/.claude/agents/`) are the primary source of truth for workflow. This document is a secondary human-readable reference.

For label definitions and colours, see [labels.md](labels.md).

## Issue lifecycle

```
Created  -->  needs-refining  -->  agent-approved  -->  PR merged  -->  Closed
              (review phase)      (implementation)
```

### Refinement phase

A `needs-refining` issue has one **status** label (why it's blocked) and one **owner** label (who should act next):

- `needs-refining` + `status:ideation` + `owner:lucas42` -- vague idea; revisit when relevant.
- `needs-refining` + `status:needs-design` + `owner:lucos-architect` -- clear goal; architect fleshes out the approach.
- `needs-refining` + `status:awaiting-decision` + `owner:lucas42` -- options laid out; lucas42 picks one.

**Principle**: only route to `owner:lucas42` when his input is genuinely needed. If an agent can do preparatory work first, route it there.

### Implementation phase

An `agent-approved` issue has an **owner** label (who will implement it) and a **priority** label:

- `agent-approved` + `owner:lucos-developer` + `priority:high` -- ready for the developer, high priority.
- `agent-approved` + `owner:lucos-system-administrator` + `priority:medium` -- infra work, standard priority.
- `agent-approved` + `owner:lucos-developer` + `status:blocked` -- ready but waiting on a dependency.

### Implementation assignment

When lucos-issue-manager marks an issue `agent-approved`, it also assigns who will implement it:

- **Default**: `owner:lucos-developer`
- **ADRs and architectural documentation** (writing ADRs, documenting conventions): `owner:lucos-architect`
- **Purely infrastructure** (Docker, deployment, server setup): `owner:lucos-system-administrator`
- **Purely monitoring/logging/pipelines**: `owner:lucos-site-reliability`
- **Purely security** (auth setup, vulnerability remediation): `owner:lucos-security`
- **Mixed work**: `owner:lucos-developer` (ensure specialist reviewed first)
- **If unclear**: `owner:lucos-developer`

### Priority

Issues are picked up in priority order: `priority:high` first, then `priority:medium`, then `priority:low`. Within the same priority level, oldest first.

Issues without a priority label have not yet been prioritised -- distinct from `priority:medium`.

### Blocked issues

`agent-approved` + `status:blocked` means the issue is well-defined but waiting on another issue. It should not be picked up. When the blocking issue is closed, lucos-issue-manager removes `status:blocked`.

## Specialist follow-up routing

Some issues need specialist review after the primary owner finishes but before `agent-approved`. Two domains:

### SRE follow-up

Issues touching **monitoring, logging, observability, or reliability** get re-routed to `owner:lucos-site-reliability` for SRE review after the primary owner's work is complete. Also applies mid-lifecycle if reliability concerns are raised in comments.

### Security follow-up

Issues touching **authentication, authorisation, data protection, or secret management** get re-routed to `owner:lucos-security` for security review after the primary owner's work is complete. Also applies mid-lifecycle if security concerns are raised in comments.

## Label management

**lucos-issue-manager is the sole agent responsible for managing labels.** No other agent adds, removes, or changes labels.

### Review agents

When a review agent (architect, sysadmin, SRE, security, code-reviewer) picks up a `needs-refining` issue:

1. Read the full issue and comments.
2. Do the work (design proposal, security assessment, etc.).
3. Post a summary comment describing what was done and the recommended next step.
4. **Do not touch labels.**

### Implementation agents

When an implementation agent (developer, architect, sysadmin, SRE, security) picks up an `agent-approved` issue:

1. Post a starting comment explaining their approach.
2. Create a branch, implement, test, open a PR with `Closes #N`.
3. The PR merge automatically closes the issue.

### lucos-issue-manager transitions

On each triage pass:

- **Refinement complete**: remove `needs-refining`, `status:*`, review-phase `owner:*`; add `agent-approved`, implementation `owner:*`, `priority:*`.
- **Route to next specialist**: update `status:*` and `owner:*`.
- **Incomplete work**: leave labels as-is or comment asking for clarification.
- **Blocked dependency resolved**: remove `status:blocked`.

### Detecting completed agent work

When an agent's comment is the most recent activity on an issue with no subsequent reply, treat it as a signal the agent has finished. Review the comment and transition accordingly.

### Reactions as approval

A +1 reaction from lucas42 on a comment counts as approval of that comment's recommendations. If the +1'd comment contains a design proposal, treat the design as approved. If it lays out options with a recommendation, treat the +1 as agreement with the recommended option.

## Reviewing vs implementing

Agents respond to two distinct prompts:

- **"review your issues"** -- reviewing: runs `get-issues-for-persona --review <persona>`, which returns only `needs-refining` issues. The agent provides design input, specialist review, or discussion.
- **"implement issue {url}"** -- implementing: the dispatcher runs `get-next-implementation-issue` to find the single highest-priority `agent-approved`, non-blocked issue across all repos. It reads the `owner:*` label to determine which persona to dispatch, then passes the specific issue URL. The agent implements the issue, opens a PR, then stops.

All implementation agents run in the same sandbox, so the dispatcher controls sequencing by picking one issue at a time. This avoids filesystem conflicts and keeps changes small, focused, and easy to debug.

See [agent-prompts.md](agent-prompts.md) for the full prompt reference.
