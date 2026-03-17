# Incident: [Short title]

| Field | Value |
|---|---|
| **Date** | YYYY-MM-DD |
| **Duration** | ~X minutes / X hours (HH:MM UTC to HH:MM UTC) |
| **Severity** | Complete outage / Partial degradation / Data risk / Security |
| **Services affected** | list of affected services |
| **Detected by** | monitoring alert / user report / ops check / etc. |

---

## Summary

A 2–3 sentence description of what happened, what the impact was, and how it was resolved. Written for someone who wasn't there.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| HH:MM | First event |
| HH:MM | Next event |
| HH:MM | Service restored / incident resolved |

---

## Analysis

Most incidents don't have a single root cause — they are the result of several conditions coinciding. Focus on the contributing factors that made the incident possible, made it worse, or delayed detection. Each stage or factor gets its own sub-heading.

For a simple incident with a clear single cause, a single "Root Cause" sub-heading is fine. For incidents that cascaded through multiple stages, use a separate sub-heading per stage and describe the contributing factors within each.

> **Note on cross-repo references:** All incident reports live in the `lucos` repo. Always use the fully-qualified format `lucas42/repo_name#N` (e.g. `lucas42/lucos_monitoring#54`) for issues and PRs in other repositories — bare `#N` references will be misinterpreted as links to `lucos` issues.

### [Stage or factor name]

What happened at this stage, and what conditions made it possible or worse.

---

## What Was Tried That Didn't Work

List diagnostic steps, attempted fixes, or hypotheses that turned out to be wrong. This section is genuinely useful — it prevents future responders from wasting time on dead ends.

If everything tried during the incident worked on the first attempt, note that here and delete this section.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Description of action | lucas42/repo_name#N | Open / In progress / Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[ ] Yes — see note below.

If yes: describe what category of sensitive information was involved and what was redacted from this report. Do not include the actual sensitive data.
