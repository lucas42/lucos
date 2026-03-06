# Incident Reports

This directory contains post-incident reports for lucos service incidents.

## What qualifies as an incident?

An event is worth a report if any of the following apply:

- A service was unavailable or significantly degraded for more than 15 minutes
- There was a risk of data loss or data integrity compromise
- A security-relevant misconfiguration was live in production
- The incident caused multiple follow-up issues to be raised
- Something happened that surprised us and we want to make sure we learn from it

Not every blip needs a report. A 2-minute container restart after a config push is not an incident. A 13-hour build break followed by a batch deployment that took down a service absolutely is.

## File naming

`YYYY-MM-DD-short-description.md` — date is when the incident occurred (not when the report was written).

Example: `2026-03-04-lucos-photos-build-break.md`

## Report structure

Use `TEMPLATE.md` as the starting point for new reports.

## Review process

Incident reports go through PRs for review before being merged to `main`. This:

- Gives others a chance to add context or correct the timeline
- Ensures the lessons learned section has been considered by more than one pair of eyes
- Creates an audit trail of when the report was written and by whom

## Security redaction checklist

Before merging, verify the report does not contain:

- Credentials, tokens, API keys, or passwords (even partial ones)
- Stack traces that expose internal file system paths or library internals beyond what's useful
- Internal hostnames or IP addresses not already publicly known
- Precise timing of when a security misconfiguration was live (if that information could aid an attacker)

If any of the above were relevant to the incident, include a note in the report's **Sensitive Findings** section explaining what was redacted and why.
