# Agent Prompts: Triggering Work Queues

How to get each agent to discover and work through its own backlog.

There are two distinct types of work: **triaging** (lucos-issue-manager assessing issues, applying labels, consulting other agents inline, and routing to the right owner) and **implementing** (writing code, opening PRs). Triaging uses a dedicated triage script; implementing uses a global dispatcher that picks the highest-priority issue across all repos and routes it to the correct persona.

---

## Routine dispatch (`/routine`)

Dispatches all agents in sequential phases for ops checks, triage, and summary. The dispatcher must wait for each phase to fully complete before starting the next.

```
/routine
```

This is a custom Claude Code skill (defined in `~/.claude/skills/routine/SKILL.md`).

**Phase 1** (parallel): Ops checks + PR review

- `lucos-code-reviewer` -- "review any open PRs"
- `lucos-security` -- "run your ops checks"
- `lucos-system-administrator` -- "run your ops checks"
- `lucos-site-reliability` -- "run your ops checks"

Ops checks run first so that any issues they raise are available for triage in Phase 2. PR review is independent of the issue pipeline and runs in parallel.

**Phase 2** (sequential, after Phase 1): Triage with inline agent consultation

- `lucos-issue-manager` -- "triage your issues"

The issue manager handles the full triage lifecycle in a single pass. When an issue needs input from another agent (e.g. architect, SRE, security), the issue manager messages that agent directly during triage, waits for their response, then re-assesses the issue. This continues until the issue is either `agent-approved` or needs input from lucas42.

**Phase 3** (after Phase 2): Summary for the user

The dispatcher compiles a prioritised list of issues that need the user's attention (any open issue with `owner:lucas42`).

---

## Triaging issues

Only the issue manager triages issues. Other agents provide input when consulted by the issue manager during triage.

### lucos-issue-manager (triage)

Triages issues (unlabelled, updated since last triage, or routed back to it). Uses its own triage script (`get-issues-for-triage`). When an issue needs specialist input, the issue manager messages the relevant agent directly via SendMessage, waits for their response, then re-assesses.

```
lucos-issue-manager, triage your issues
```

### Other agents (consulted during triage)

Agents like `lucos-architect`, `lucos-security`, `lucos-site-reliability`, `lucos-system-administrator`, and `lucos-developer` do not have their own issue review queues. Instead, the issue manager messages them directly during triage when their input is needed on a specific issue. The agent reads the issue, posts a comment with their input, and messages the issue manager back.

### lucos-code-reviewer

Reviews open PRs. Triggered during Phase 1 of `/routine` or on demand.

```
lucos-code-reviewer, review any open PRs
```

---

## Ops checks

Agents with operational responsibilities run proactive checks when prompted:

```
lucos-security, run your ops checks
lucos-system-administrator, run your ops checks
lucos-site-reliability, run your ops checks
```

These are typically run in Phase 1 of `/routine` so any issues they raise are available for triage.

---

## Implementing issues

Implementation is driven by the dispatcher, not individual agents. A single global script finds the highest-priority issue across all repos and the dispatcher routes it to the correct persona.

```
/next
```

This is a custom Claude Code skill (defined in `~/.claude/skills/next/SKILL.md`).

The dispatcher will:
1. Run `get-next-implementation-issue`, which searches across all repos for the single highest-priority `agent-approved`, non-blocked issue with an `owner:*` label.
2. Read the `owner:*` label to determine which persona to dispatch (e.g. `owner:lucos-developer` -> launch `lucos-developer`).
3. Pass the specific issue URL to the persona (e.g. "implement issue https://github.com/lucas42/lucos_photos/issues/42").
4. The persona posts a starting comment, implements, and opens a PR.
5. The implementation teammate then drives its own **review loop** with `lucos-code-reviewer` (see below).

### Review loop

After opening a PR, the implementation teammate (not the dispatcher) drives a back-and-forth with the code reviewer:

1. The implementation teammate messages `lucos-code-reviewer` to review the PR.
2. If the reviewer **approves**, the loop is done.
3. If the reviewer **requests changes**, the implementation teammate addresses the feedback, pushes fixes, and requests another review.
4. Go back to step 1.

This loop runs for up to **5 iterations**. If the PR is still not approved after 5 rounds, the teammate stops and flags it for lucas42 to review.

The full procedure is documented in `~/.claude/pr-review-loop.md`.

### Why the dispatcher picks the issue

All implementation agents run in the same sandbox. If multiple personas were dispatched to different repos simultaneously, their local filesystem writes could conflict. The dispatcher controls sequencing by picking one issue at a time.

### Notes

- `lucos-developer` is the default implementation persona. Most `agent-approved` issues will be assigned to it.
- `lucos-architect` handles ADR and architectural documentation issues.
- `lucos-system-administrator` handles purely infrastructure issues.
- `lucos-site-reliability` handles monitoring, logging, pipeline, and incident management issues.
- `lucos-security` handles purely security issues.
- `lucos-issue-manager` handles workflow and process documentation issues.
- The "implement" prompt is deliberately separate from triage to avoid multiple simultaneous changes that are hard to debug and expensive on credits.
