# Agent Prompts: Triggering Work Queues

How to get each agent to discover and work through its own backlog.

---

## All agents at once

Launches all six agents in parallel, each working through its own backlog concurrently.

```
all agents, work on your issues
```

This is equivalent to running each individual agent prompt below simultaneously. The dispatcher will launch all six agents using parallel Task tool calls -- no sequential execution, no clarification needed.

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
