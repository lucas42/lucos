# Agent Prompts: Triggering Work Queues

How to get each agent to discover and work through its own backlog.

There are two distinct types of work: **reviewing** (design input, specialist review, triage) and **implementing** (writing code, opening PRs). Each has its own prompt and its own script invocation, returning non-overlapping sets of issues.

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

Implementation uses a separate prompt to ensure only one issue is worked on at a time. Each agent implements `agent-approved` issues assigned to it (via `get-issues-for-persona --implement`).

Tell a specific agent to implement the single highest-priority issue in its queue:

```
lucos-developer, implement your next issue
```

```
lucos-system-administrator, implement your next issue
```

```
lucos-site-reliability, implement your next issue
```

```
lucos-security, implement your next issue
```

The agent will:
1. Run `get-issues-for-persona --implement <persona>`, which returns the single highest-priority `agent-approved`, non-blocked issue.
2. Post a starting comment, implement, and open a PR.
3. Stop after that one issue.

### Notes

- `lucos-developer` is the default implementation persona. Most `agent-approved` issues will be assigned to it.
- The "implement" prompt is deliberately separate from "review" to avoid multiple simultaneous changes that are hard to debug and expensive on credits.
- The `--review` and `--implement` flags return non-overlapping sets of issues, so there is no ambiguity about what an agent should do with the issues it receives.
