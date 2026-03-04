# Agent Prompts: Triggering Work Queues

How to get each agent to discover and work through its own backlog.

There are two distinct types of work: **reviewing** (design input, specialist review, triage) and **implementing** (writing code, opening PRs). Reviewing uses a per-persona script; implementing uses a global dispatcher that picks the highest-priority issue across all repos and routes it to the correct persona.

---

## All agents, review

Dispatches all agents in three sequential phases for review/triage work. The dispatcher must wait for each phase to fully complete before starting the next.

```
all agents, review your issues
```

**Phase 1** (parallel): `lucos-issue-manager` + `lucos-code-reviewer`

The issue manager runs first to assign `owner:` labels to unowned issues, so that Phase 2 agents pick up fresh work. The code reviewer is independent of the issue pipeline and runs in parallel with it.

**Phase 2** (parallel, after Phase 1): `lucos-architect` + `lucos-system-administrator` + `lucos-security` + `lucos-site-reliability`

The four specialist agents review their assigned backlogs. They post design proposals, comments, or assessments, which may leave issues needing reassignment.

**Phase 3** (after Phase 2): `lucos-issue-manager` again

A second pass by the issue manager to review anything Phase 2 agents touched, reassign or transition labels, and tidy up issues left in an intermediate state.

---

## Reviewing issues

Each agent reviews `needs-refining` issues assigned to it (via `get-issues-for-persona --review`).

### lucos-issue-manager

Triages issues (unlabelled, updated since last review, or routed back to it). Uses its own triage script rather than `get-issues-for-persona`.

```
lucos-issue-manager, review your issues
```

### lucos-code-reviewer

Reviews open PRs and reviews its assigned issues.

```
lucos-code-reviewer, review your issues
```

### lucos-architect

Reviews `needs-refining` issues labelled `owner:lucos-architect`.

```
lucos-architect, review your issues
```

### lucos-system-administrator

Reviews `needs-refining` issues labelled `owner:lucos-system-administrator`.

```
lucos-system-administrator, review your issues
```

### lucos-security

Reviews dependabot alerts, then reviews `needs-refining` issues labelled `owner:lucos-security`.

```
lucos-security, review your issues
```

### lucos-site-reliability

Reviews `needs-refining` issues labelled `owner:lucos-site-reliability`.

```
lucos-site-reliability, review your issues
```

---

## Implementing issues

Implementation is driven by the dispatcher, not individual agents. A single global script finds the highest-priority issue across all repos and the dispatcher routes it to the correct persona.

```
implement the next issue
```

The dispatcher will:
1. Run `get-next-implementation-issue`, which searches across all repos for the single highest-priority `agent-approved`, non-blocked issue with an `owner:*` label.
2. Read the `owner:*` label to determine which persona to dispatch (e.g. `owner:lucos-developer` → launch `lucos-developer`).
3. Pass the specific issue URL to the persona (e.g. "implement issue https://github.com/lucas42/lucos_photos/issues/42").
4. The persona posts a starting comment, implements, and opens a PR.
5. After the persona finishes, the dispatcher checks for a new PR and launches `lucos-code-reviewer` to review it if one was created.

### Why the dispatcher picks the issue

All implementation agents run in the same sandbox. If multiple personas were dispatched to different repos simultaneously, their local filesystem writes could conflict. The dispatcher controls sequencing by picking one issue at a time.

### Notes

- `lucos-developer` is the default implementation persona. Most `agent-approved` issues will be assigned to it.
- `lucos-architect` handles ADR and architectural documentation issues. These are issues whose primary deliverable is writing an ADR or documenting a convention.
- The "implement" prompt is deliberately separate from "review" to avoid multiple simultaneous changes that are hard to debug and expensive on credits.
