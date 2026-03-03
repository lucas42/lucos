# Agent Prompts: Triggering Work Queues

How to get each agent to discover and work through its own backlog.

---

## All agents at once

Dispatches all agents in three sequential phases. The dispatcher must wait for each phase to fully complete before starting the next.

```
all agents, work on your issues
```

**Phase 1** (parallel): `lucos-issue-manager` + `lucos-code-reviewer`

The issue manager runs first to assign `owner:` labels to unowned issues, so that Phase 2 agents pick up fresh work. The code reviewer is independent of the issue pipeline and runs in parallel with it.

**Phase 2** (parallel, after Phase 1): `lucos-architect` + `lucos-system-administrator` + `lucos-security` + `lucos-site-reliability`

The four specialist agents work through their assigned backlogs. They often add comments or partial work rather than closing issues outright, which may leave issues needing reassignment.

**Phase 3** (after Phase 2): `lucos-issue-manager` again

A second pass by the issue manager to review anything Phase 2 agents touched, reassign or transition labels, and tidy up issues left in an intermediate state.

---

## lucos-issue-manager

Triages issues (unlabelled, updated since last review, or routed back to it).

```
lucos-issue-manager, work on your issues
```

## lucos-code-reviewer

Reviews open PRs and works on its assigned issues.

```
lucos-code-reviewer, work on your issues
```

This runs both PR discovery and issue discovery in a single session.

## lucos-architect

Works on issues labelled `owner:lucos-architect`.

```
lucos-architect, work on your issues
```

## lucos-system-administrator

Works on issues labelled `owner:lucos-system-administrator`.

```
lucos-system-administrator, work on your issues
```

## lucos-security

Works on issues labelled `owner:lucos-security`.

```
lucos-security, work on your issues
```

## lucos-site-reliability

Works on issues labelled `owner:lucos-site-reliability`.

```
lucos-site-reliability, work on your issues
```
