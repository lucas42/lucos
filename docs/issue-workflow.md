# Issue Workflow

How issues move through the lucos pipeline from creation to completion.

The persona instruction files (under `~/.claude/agents/`) are the primary source of truth for workflow. This document is a secondary human-readable reference.

For label definitions and colours, see [labels.md](labels.md).

## Issue lifecycle

```
Created  -->  needs-refining  -->  agent-approved  -->  PR merged  -->  Closed
           (triage + consultation)   (implementation)
```

### Refinement phase

A `needs-refining` issue has one **status** label (why it's blocked), one **owner** label (who should act next), and a **priority** label:

- `needs-refining` + `status:ideation` + `owner:lucas42` + `priority:low` -- vague idea; revisit when relevant.
- `needs-refining` + `status:needs-design` + `owner:lucos-architect` + `priority:medium` -- clear goal; architect fleshes out the approach.
- `needs-refining` + `status:awaiting-decision` + `owner:lucas42` + `priority:high` -- options laid out; lucas42 picks one.

**Principle**: only route to `owner:lucas42` when his input is genuinely needed. If an agent can do preparatory work first, route it there.

### Implementation phase

An `agent-approved` issue has an **owner** label (who will implement it) and a **priority** label:

- `agent-approved` + `owner:lucos-developer` + `priority:high` -- ready for the developer, high priority.
- `agent-approved` + `owner:lucos-system-administrator` + `priority:medium` -- infra work, standard priority.
- `agent-approved` + `owner:lucos-developer` + `status:blocked` -- ready but waiting on a dependency.

### Priority across the lifecycle

Priority labels are assigned to **all issues during triage**, not just `agent-approved` ones. A `needs-refining` issue also gets a `priority:*` label so that refinement work is prioritised too.

When lucas42 gives input on an issue (e.g. a comment, a decision, or a reaction), the priority is re-assessed -- the scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

**Priority override rules:**
- lucas42's explicit priority calls override the strategic priorities framework (lucas42 is the repo owner and has final say).
- Priority calls from others (including agents) are considered within the context of the strategic priorities defined in [priorities.md](priorities.md) -- they do not automatically override.

### Implementation assignment

When lucos-issue-manager marks an issue `agent-approved`, it also assigns who will implement it:

- **Default**: `owner:lucos-developer`
- **ADRs and architectural documentation** (writing ADRs, documenting conventions): `owner:lucos-architect`
- **Purely infrastructure** (Docker, deployment, server setup): `owner:lucos-system-administrator`
- **Purely monitoring/logging/pipelines**: `owner:lucos-site-reliability`
- **Incident management** (incident response, reporting, post-mortems, tracking): `owner:lucos-site-reliability`
- **Purely security** (auth setup, vulnerability remediation): `owner:lucos-security`
- **Workflow and process documentation** (issue conventions, label conventions, triage process, agent workflow docs): `owner:lucos-issue-manager`
- **Mixed work**: `owner:lucos-developer` (ensure specialist reviewed first)
- **If unclear**: `owner:lucos-developer`

### Priority

`get-next-implementation-issue` searches all repositories for `agent-approved`, non-blocked issues and sorts by priority label (`priority:critical` > `priority:high` > `priority:medium` > `priority:low` > unprioritised), then oldest first within each band.

Issues without a priority label have not yet been prioritised -- distinct from `priority:medium`.

For strategic guidance on which areas of work map to which priority level, see [priorities.md](priorities.md).

### Blocked issues

`agent-approved` + `status:blocked` means the issue is well-defined but waiting on another issue. It should not be picked up. When the blocking issue is closed, lucos-issue-manager removes `status:blocked`.

## Specialist follow-up routing

Some issues need specialist review after the primary agent's input, but before `agent-approved`. Two domains:

### SRE follow-up

Issues touching **monitoring, logging, observability, reliability, or incident management** (incident response, reporting, post-mortems, tracking) get SRE input before approval. During triage, the issue manager consults the primary agent first (e.g. architect for design), then consults `lucos-site-reliability` sequentially so the SRE sees the primary agent's comment. Also applies mid-lifecycle if reliability or incident management concerns are raised in comments.

### Security follow-up

Issues touching **authentication, authorisation, data protection, or secret management** get security input before approval. During triage, the issue manager consults the primary agent first, then consults `lucos-security` sequentially so the security agent sees the primary agent's comment. Also applies mid-lifecycle if security concerns are raised in comments.

## Label management

**lucos-issue-manager is the sole agent responsible for managing labels.** No other agent adds, removes, or changes labels.

### Consulted agents

When an agent (architect, sysadmin, SRE, security, code-reviewer) is consulted by the issue manager during triage:

1. Read the full issue and comments.
2. Do the work (design proposal, security assessment, etc.).
3. Post a summary comment on the issue describing what was done and the recommended next step.
4. Message the issue manager back to confirm they've posted their input.
5. **Do not touch labels.**

### Implementation agents

When an implementation agent (developer, architect, sysadmin, SRE, security) picks up an `agent-approved` issue:

1. Post a starting comment explaining their approach.
2. Create a branch, implement, test, open a PR with `Closes #N`.
3. The PR merge automatically closes the issue.

### lucos-issue-manager transitions

Most agent consultation happens inline during triage (via SendMessage). The issue manager consults agents, waits for their response, and transitions labels within a single triage pass.

For issues that were labelled with `owner:` and `needs-refining` in a previous session (before the inline flow), the issue manager detects completed agent work: if an agent's comment is the most recent activity with no subsequent reply, treat it as their completed input and transition accordingly.

Transitions that apply between triage passes:

- **Blocked dependency resolved**: remove `status:blocked` to make the issue available for pickup.
- **A PR with a `Closes #N` keyword has been merged**: the issue is automatically closed; no label action needed.

### Reactions as approval

A +1 reaction from lucas42 on a comment counts as approval of that comment's recommendations. If the +1'd comment contains a design proposal, treat the design as approved. If it lays out options with a recommendation, treat the +1 as agreement with the recommended option.

## Ops check duplicate prevention

Personas that run ops checks (lucos-security, lucos-site-reliability, lucos-system-administrator) must always search for existing open issues before creating new ones. The rules are:

1. **If an open issue already exists** for the same problem: take no action. Do not create a duplicate.
2. **If an open issue exists but new information has been discovered** (e.g. new symptoms, a higher severity, additional affected files): add a comment to the existing issue with the new information.
3. **If no open issue exists**: create a new one following the check-specific instructions.

This prevents duplicate tickets from accumulating when multiple ops check runs detect the same underlying problem.

## Audit-finding issues

Issues with the `audit-finding` label are created automatically by the `lucos_repos` audit tool when a repository convention fails. See [ADR-0002](https://github.com/lucas42/lucos_repos/blob/main/docs/adr/0002-audit-issue-lifecycle.md) for the full design.

### How the audit tool works

On each sweep (up to every 6 hours), for each repo + convention pair:

| Convention result | Open issue exists? | Action |
|---|---|---|
| Pass | No | Do nothing |
| Pass | Yes | Do nothing |
| Fail | Yes (open) | Do nothing |
| Fail | No (none or only closed) | Create a new issue |

The audit tool **only creates issues**. It never closes, reopens, or comments on issues. The issue tracker is a notification mechanism, not the canonical record of compliance.

### Triage implications

- Triage `audit-finding` issues normally -- approve, route, and prioritise them like any other issue.
- Issues are closed via the normal workflow (PR merge with closing keyword, or manual close). The audit tool will not close them.
- If an issue is closed but the convention still fails, the next sweep creates a **new** issue (not a reopened one).
- There is no suppression mechanism. If a convention does not apply to a repo, the convention's check function must encode that logic.

## Triaging vs implementing

Agents respond to distinct prompts depending on their role:

- **"triage your issues"** -- triaging (lucos-issue-manager only): runs `get-issues-for-triage`, which returns unlabelled issues, issues with recent activity, or issues routed back to the issue manager. The issue manager assesses clarity, applies labels, and drives issues toward `agent-approved` or `owner:lucas42`. When an issue needs specialist input (design, security, reliability, etc.), the issue manager messages the relevant agent directly during triage, waits for their response, then re-assesses. This inline consultation replaces the previous separate review phase.
- **"run your ops checks"** -- ops checks (security, SRE, sysadmin): each agent runs its proactive operational checks (dependabot alerts, monitoring status, container health, etc.) and raises issues for anything found. These run before triage so newly raised issues are included in the triage pass.
- **"implement issue {url}"** -- implementing: the dispatcher runs `get-next-implementation-issue` to find the highest-priority `agent-approved`, non-blocked issue across all repos. It reads the `owner:*` label to determine which persona to dispatch, then passes the specific issue URL. The agent implements the issue, opens a PR, then stops.

All implementation agents run in the same sandbox, so the dispatcher controls sequencing by picking one issue at a time. This avoids filesystem conflicts and keeps changes small, focused, and easy to debug.

See [agent-prompts.md](agent-prompts.md) for the full prompt reference.
