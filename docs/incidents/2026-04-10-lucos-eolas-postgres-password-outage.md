# Incident: lucos_eolas production outage — missing POSTGRES_PASSWORD in DATABASES config

| Field | Value |
|---|---|
| **Date** | 2026-04-10 |
| **Duration** | ~16 minutes (14:27 UTC to ~14:43 UTC) |
| **Severity** | Complete outage |
| **Services affected** | eolas.l42.eu (lucos_eolas) |
| **Detected by** | User report; confirmed via monitoring (9 consecutive `/_info` 502 failures) |

---

## Summary

PR lucas42/lucos_eolas#162 ("Add CI test suite") switched the test database to `trust` auth and removed `POSTGRES_PASSWORD` from Django's `DATABASES` config in `settings.py`. The intent was that tests no longer needed a password; however, the production Postgres instance still requires one. The container crash-looped immediately after the CI deploy with `fe_sendauth: no password supplied`. A one-line fix (lucas42/lucos_eolas#163) restoring the `PASSWORD` field was merged and deployed, restoring service.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 14:27:34 | PR lucas42/lucos_eolas#162 merged; CI auto-deploy begins |
| ~14:28 | New container deployed; crash-loops immediately on DB connection attempt |
| ~14:29 | Monitoring begins accumulating `/_info` 502 failures |
| ~14:35 | User reports production is broken |
| 14:38 | SRE picks up incident; monitoring confirms 9 consecutive 502s |
| 14:38 | Container logs pulled from avalon — `fe_sendauth: no password supplied` confirmed |
| 14:39 | `settings.py` diff reviewed; root cause identified |
| 14:42 | Fix committed (restore `'PASSWORD': os.environ.get('POSTGRES_PASSWORD', '')`) |
| 14:43 | PR lucas42/lucos_eolas#163 opened |
| ~14:43 | PR approved and merged; CI auto-deploy begins |
| ~14:50 | Service restored |

---

## Analysis

### Root cause: POSTGRES_PASSWORD removed from DATABASES config

PR lucas42/lucos_eolas#162 aligned the test database setup with the `lucos_contacts` pattern, which uses `POSTGRES_HOST_AUTH_METHOD=trust` on the test `db` service. Under `trust` auth, no password is required. The developer removed `'PASSWORD': os.environ['POSTGRES_PASSWORD']` from `settings.py`'s `DATABASES` config as part of this change.

This was valid for the test path — the test `db` container accepts connections without a password — but the production Postgres instance does not use `trust` auth. `POSTGRES_PASSWORD` was present in the production container's environment (supplied by lucos_creds), but Django was no longer reading it into the connection config, so every connection attempt failed.

### Why it wasn't caught

There was no test exercising the production DB connection path with auth enabled. The CI suite runs against a `trust`-auth database, so the missing `PASSWORD` field caused no CI failures. The regression only surfaced on the production deploy.

---

## What Was Tried That Didn't Work

Everything worked on the first attempt. Root cause was identified from container logs within 1 minute of investigation start.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Restore POSTGRES_PASSWORD in DATABASES config | lucas42/lucos_eolas#163 | Done |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.
