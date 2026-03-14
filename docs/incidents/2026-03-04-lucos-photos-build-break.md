# Incident: lucos_photos production outage after batch deployment following 13-hour CI break

| Field | Value |
|---|---|
| **Date** | 2026-03-04 (deployment failure occurred 2026-03-05 ~01:40 UTC) |
| **Duration** | ~61 minutes (service down ~01:40 UTC to 02:11 UTC on 2026-03-05) |
| **Severity** | Complete service outage (API container failed to start) |
| **Services affected** | lucos_photos (photos.l42.eu) |
| **Detected by** | lucos-site-reliability ops check |

---

## Summary

A CI build failure introduced by lucas42/lucos_photos#54 went undetected for ~13 hours, during which six further PRs were merged. When the build was finally fixed by lucas42/lucos_photos#69 and lucas42/lucos_photos#70, all accumulated changes deployed to production at once. One of those changes (lucas42/lucos_photos#59) had renamed a module-level export in a shared library without updating `alembic/env.py`. The `ImportError` crashed the API container on startup, taking the service down for 61 minutes until lucas42/lucos_photos#72 was merged and the fix deployed.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-03-04 11:54 | lucas42/lucos_photos#54 merged. Introduces `libgl1-mesa-glx` as a dependency — a package that no longer exists in Debian Trixie. CI build begins failing silently. |
| 2026-03-04 12:15–12:41 | lucas42/lucos_photos#55, #56, #57, #58, #59 merged (health checks, metrics, GET endpoints, `pg_isready` retry loop, `engine`→`get_engine()` refactor). Build still broken; none deploy to production. |
| 2026-03-05 00:39 | lucas42/lucos_photos#68 merged (session authentication). Build still broken. |
| 2026-03-05 01:10 | lucas42/lucos_photos#69 merged. Fixes `libgl1-mesa-glx` build failure. Worker image build now passes. |
| 2026-03-05 01:39 | lucas42/lucos_photos#70 merged. Fixes second build failure (missing `gcc`/`g++`). Both API and worker builds pass. First deployment in ~13 hours fires. |
| 2026-03-05 ~01:40 | Service goes down. API container starts, runs `alembic upgrade head`, crashes with `ImportError: cannot import name 'engine' from 'lucos_photos_common.database'`. Container exits. Service unavailable. |
| 2026-03-05 02:09 | lucas42/lucos_photos#71 raised by lucos-site-reliability with root cause diagnosis. |
| 2026-03-05 02:11 | lucas42/lucos_photos#72 merged. `alembic/env.py` updated to call `get_engine()`. API container starts successfully. Service restored. |

---

## Root Cause

lucas42/lucos_photos#59 correctly replaced the module-level `engine` instance in `shared/lucos_photos_common/database.py` with a `get_engine()` function to avoid import-time side effects. However, `api/alembic/env.py` was not updated to match — it continued to import the now-removed name:

```python
from lucos_photos_common.database import Base, engine
```

Because `alembic/env.py` lives outside the main application tree (`api/alembic/` rather than `api/app/`), it was overlooked during the refactor. The `ImportError` was invisible in CI because the build was already broken for an unrelated reason when lucas42/lucos_photos#59 was merged — there were no working Docker builds to run the application and reveal the problem until the build was repaired 13 hours later.

---

## What Was Tried That Didn't Work

Everything tried during the immediate response worked on the first attempt. The root cause was identified from logs and the fix was straightforward.

---

## Resolution

lucas42/lucos_photos#72 updated `api/alembic/env.py` to import and call `get_engine()` instead of `engine`:

```python
from lucos_photos_common.database import Base, get_engine

# offline migrations
url=get_engine().url

# online migrations
with get_engine().connect() as connection:
```

The PR also removed unused imports (`pool`, `Engine`) left over from the original Alembic scaffold.

---

## Contributing Factors

### Factor 1: Build failure went undetected for ~13 hours

When lucas42/lucos_photos#54 was merged at 11:54 UTC on 2026-03-04, the CI build began failing. This was not caught or acted on until the following day. During those 13 hours, six further PRs were merged on top of the broken build. When the build was fixed, all six deployed simultaneously — making it harder to identify which change was responsible and significantly amplifying blast radius compared to six incremental deployments.

### Factor 2: Automated code review did not catch the missed `alembic/env.py` update

The code review on lucas42/lucos_photos#59 approved the `engine`→`get_engine()` refactor without flagging that `alembic/env.py` still referenced the old name. A complete review of a shared-module rename should include searching all callsites of the removed export across the full repository. `alembic/env.py` is scaffolded boilerplate that is rarely touched and easy to overlook.

### Factor 3: No static analysis in CI

There were no tests or linting steps in CI that would have caught a broken import in `alembic/env.py` before deployment. A `python -m py_compile api/alembic/env.py` step, or a `ruff`/`pyflakes` pass over the full `api/` tree, would have surfaced the `ImportError` as a build failure rather than a runtime crash in production.

### Factor 4: Alembic files outside normal test surface

The Alembic migration environment (`alembic/env.py`, `alembic/versions/`) is not exercised by the API test suite. It is only executed at container startup. Import errors in migration code are therefore invisible until a deployment occurs.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Alert on sustained CI build failure | To be raised | Open |
| Add import-time smoke tests for Alembic config in CI | To be raised | Open |
| Run ruff/pyflakes over full repo in CI | To be raised | Open |
| Treat shared-module renames as requiring repo-wide grep | Captured in SRE agent memory | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
