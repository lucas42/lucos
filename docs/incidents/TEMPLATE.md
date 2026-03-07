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

## Root Cause

Explain the technical root cause in enough detail that someone unfamiliar with the code could understand it. Include relevant code, configuration, or architecture context.

---

## What Was Tried That Didn't Work

List diagnostic steps, attempted fixes, or hypotheses that turned out to be wrong. This section is genuinely useful — it prevents future responders from wasting time on dead ends.

If everything tried during the incident worked on the first attempt, note that here and delete this section.

---

## Resolution

Describe the fix applied to restore service. Reference the specific PR or commit if applicable.

---

## Contributing Factors

Describe any conditions that made the incident worse, delayed detection, or increased blast radius. Each factor should have its own sub-heading.

### Factor 1: [Name]

Explanation.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Description of action | #N | Open / In progress / Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[ ] Yes — see note below.

If yes: describe what category of sensitive information was involved and what was redacted from this report. Do not include the actual sensitive data.
