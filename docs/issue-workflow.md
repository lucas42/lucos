# Issue Workflow

How issues move through the lucos pipeline from creation to completion.

The persona instruction files (under `~/.claude/agents/`) are the primary source of truth for workflow. This document is a secondary human-readable reference.

Issue metadata is held on the [**lucOS Issue Prioritisation** project board](https://github.com/users/lucas42/projects/8) via three custom fields (Status, Priority, Owner). For field option definitions, see the [Project Field Reference](labels.md).

## Issue lifecycle

```
Created  -->  Needs Triage  -->  Ideation / Awaiting Decision  -->  Ready  -->  PR merged  -->  Done
                  (triage)         (refinement + consultation)    (implementation)
```

`Needs Triage` is set automatically when an issue lands on the project board. Triage moves it to one of the other Status values within a single pass.

### Refinement phase

Refinement covers Ideation, Awaiting Decision, and (transiently) the inline-consultation period between agents. Every refinement-phase issue has a **Status**, an **Owner**, and a **Priority** on the project board:

- Status = Ideation, Owner = lucas42, Priority = Low — vague idea; revisit when relevant.
- Status = Awaiting Decision, Owner = lucas42, Priority = High — options laid out; lucas42 picks one.
- Status = Ideation, Owner = lucos-architect (or other specialist), Priority = Medium — agent design work in progress, or the issue is parked while a specialist designs.

**Principle**: only route to Owner = lucas42 when his input is genuinely needed. If an agent can do preparatory work first, route it there.

### Implementation phase

A Ready issue has an Owner (who will implement it) and a Priority. Examples:

- Status = Ready, Owner = lucos-developer, Priority = High — ready for the developer, high priority.
- Status = Ready, Owner = lucos-system-administrator, Priority = Medium — infra work, standard priority.
- Status = Blocked, Owner = lucos-developer — ready but waiting on a dependency to close.

### Priority across the lifecycle

Priority is set on **every triaged issue** — including refinement-phase issues, not just Ready ones. This ensures refinement work is also prioritised so lucas42 and agents can see which questions are most urgent.

When lucas42 gives input on an issue (a comment, a decision, or a reaction), priority is re-assessed — the scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

**Priority override rules:**
- lucas42's explicit priority calls override the strategic priorities framework (lucas42 is the repo owner and has final say).
- Priority calls from others (including agents) are considered within the context of the strategic priorities defined in [priorities.md](priorities.md) — they do not automatically override.

### Implementation assignment

When lucos-issue-manager moves an issue to Status = Ready, it also sets Owner to indicate who will implement it:

- **Default**: Owner = lucos-developer
- **ADRs and architectural documentation** (writing ADRs, documenting conventions): Owner = lucos-architect
- **Purely infrastructure** (Docker, deployment, server setup): Owner = lucos-system-administrator
- **Purely monitoring / logging / pipelines**: Owner = lucos-site-reliability
- **Incident management** (incident response, reporting, post-mortems, tracking): Owner = lucos-site-reliability
- **Purely security** (auth setup, vulnerability remediation): Owner = lucos-security
- **Frontend / UX / accessibility / copywriting**: Owner = lucos-ux
- **Workflow and process documentation** (issue conventions, project field conventions, triage process, agent workflow docs): Owner = lucos-issue-manager
- **Mixed work**: Owner = lucos-developer (ensure the relevant specialist has reviewed first)
- **If unclear**: Owner = lucos-developer

### Priority ordering

`get-next-implementation-issue` reads the project board and returns the topmost item in the Ready column. Ordering is determined by board position, which lucas42 may manually adjust to override the Priority sort. Within priorities, ordering is: Critical > High > Medium > Low > unprioritised, oldest first within each band.

Items without a Priority set have not yet been prioritised — distinct from Priority = Medium.

For strategic guidance on which areas of work map to which priority level, see [priorities.md](priorities.md).

### Blocked issues

Status = Blocked means the issue is well-defined but waiting on another issue. It should not be picked up. When the blocking issue is closed, lucos-issue-manager moves it back to Status = Ready and repositions it on the board by priority.

## Specialist follow-up routing

Some issues need specialist review after the primary agent's input, but before they can move to Ready. Two domains:

### SRE follow-up

Issues touching **monitoring, logging, observability, reliability, or incident management** (incident response, reporting, post-mortems, tracking) get SRE input before approval. During triage, the issue manager consults the primary agent first (e.g. architect for design), then consults `lucos-site-reliability` sequentially so the SRE sees the primary agent's comment. Also applies mid-lifecycle if reliability or incident management concerns are raised in comments.

### Security follow-up

Issues touching **authentication, authorisation, data protection, or secret management** get security input before approval. During triage, the issue manager consults the primary agent first, then consults `lucos-security` sequentially so the security agent sees the primary agent's comment. Also applies mid-lifecycle if security concerns are raised in comments.

## Project field management

**lucos-issue-manager is the sole agent responsible for setting project board fields.** No other agent adds, removes, or changes Status, Priority, or Owner.

### Consulted agents

When an agent (architect, sysadmin, SRE, security, code-reviewer, UX) is consulted by the issue manager during triage:

1. Read the full issue and comments.
2. Do the work (design proposal, security assessment, etc.).
3. Post a summary comment on the issue describing what was done and the recommended next step.
4. Message the issue manager back to confirm they've posted their input.
5. **Do not touch project board fields.**

### Implementation agents

When an implementation agent (developer, architect, sysadmin, SRE, security, UX) picks up a Ready issue:

1. Post a starting comment explaining their approach.
2. Create a branch, implement, test, open a PR with `Closes #N`.
3. The PR merge automatically closes the issue and the project board sets Status to Done.

### lucos-issue-manager transitions

Most agent consultation happens inline during triage (via SendMessage). The issue manager consults agents, waits for their response, and updates Status within a single triage pass.

For issues left in a refinement-phase Status from a previous session, the issue manager detects completed agent work: if an agent's comment is the most recent activity with no subsequent reply, treat it as their completed input and update Status accordingly.

Transitions that apply between triage passes:

- **Blocked dependency resolved**: change Status from Blocked back to Ready to make the issue available for pickup, then reposition by priority.
- **A PR with a `Closes #N` keyword has been merged**: the issue is automatically closed and the board sets Status = Done; no manual action needed.

### Reactions as approval

A +1 reaction from lucas42 on a comment counts as approval of that comment's recommendations. If the +1'd comment contains a design proposal, treat the design as approved. If it lays out options with a recommendation, treat the +1 as agreement with the recommended option.

## Duplicate prevention

**All agents must search for existing open issues before creating new ones.** This applies to every context: ops checks, incident follow-up, code review observations, ad-hoc issue creation, and any other situation where an agent creates an issue.

The rules are:

1. **If an open issue already exists** for the same problem (on any repo — check cross-repo too): take no action. Do not create a duplicate.
2. **If an open issue exists but new information has been discovered** (e.g. new symptoms, a higher severity, additional affected files): add a comment to the existing issue with the new information.
3. **If no open issue exists**: create a new one following the relevant instructions.

Search broadly: the same problem may be filed on a different repo than you'd expect (e.g. a deploy validation issue might be on the deploy orb repo, the affected service repo, or the cross-cutting `lucos` repo). Search the org, not just the target repo.

This is especially important during **incident follow-up**, where multiple agents (SRE, architect, sysadmin, issue manager) are independently analysing the same incident and may identify the same follow-up action. To prevent this: **the SRE writing the incident report is the designated owner of filing all follow-up issues from an incident.** Other agents must not independently file follow-up issues — if they identify a follow-up action, they should message the SRE or the issue manager instead. See `references/incident-reporting.md` for the full process.

## Audit-finding issues

Issues with the `audit-finding` label are created automatically by the `lucos_repos` audit tool when a repository convention fails. `audit-finding` is the one workflow-relevant label that wasn't migrated to a project field — see the [Project Field Reference](labels.md) for the rationale. See [ADR-0002](https://github.com/lucas42/lucos_repos/blob/main/docs/adr/0002-audit-issue-lifecycle.md) for the full design.

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

- Triage `audit-finding` issues normally — set Status, Priority, Owner like any other issue.
- Issues are closed via the normal workflow (PR merge with closing keyword, or manual close). The audit tool will not close them.
- If an issue is closed but the convention still fails, the next sweep creates a **new** issue (not a reopened one).
- There is no suppression mechanism. If a convention does not apply to a repo, the convention's check function must encode that logic.

## Triaging vs implementing

Agents respond to distinct prompts depending on their role:

- **"triage your issues"** — triaging (lucos-issue-manager only): runs `get-issues-for-triage`, which reads the project board for issues that need attention (untriaged, recently commented, routed to the issue manager). The issue manager assesses clarity, sets Status / Priority / Owner, and drives issues toward Ready or Owner = lucas42. When an issue needs specialist input (design, security, reliability, etc.), the issue manager messages the relevant agent directly during triage, waits for their response, then re-assesses. This inline consultation replaces the previous separate review phase.
- **"run your ops checks"** — ops checks (security, SRE, sysadmin): each agent runs its proactive operational checks (dependabot alerts, monitoring status, container health, etc.) and raises issues for anything found. These run before triage so newly raised issues are included in the triage pass.
- **"implement issue {url}"** — implementing: the dispatcher runs `get-next-implementation-issue` to find the highest-priority Ready issue across all repos. It reads the Owner field on the project board to determine which persona to dispatch, then passes the specific issue URL. The agent implements the issue, opens a PR, then stops.

All implementation agents run in the same sandbox, so the dispatcher controls sequencing by picking one issue at a time. This avoids filesystem conflicts and keeps changes small, focused, and easy to debug.

See [agent-prompts.md](agent-prompts.md) for the full prompt reference.
