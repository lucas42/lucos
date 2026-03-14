# Incident: lucos_monitoring down — ssl_not_started on all HTTPS fetches, then crash-loop

| Field | Value |
|---|---|
| **Date** | 2026-03-13 (extended into 2026-03-14) |
| **Duration** | ~11 hours (21:54 UTC 2026-03-13 to 08:54 UTC 2026-03-14) |
| **Severity** | Complete outage — monitoring blind for ~8 hours, crash-looping for ~3 hours |
| **Services affected** | lucos_monitoring (monitoring.l42.eu) — all monitored services appeared as erroring throughout |
| **Detected by** | User report |

Source issues: [lucas42/lucos_monitoring#52](https://github.com/lucas42/lucos_monitoring/issues/52), [lucas42/lucos_monitoring#47](https://github.com/lucas42/lucos_monitoring/pull/47)

---

## Summary

A fix for an unrelated startup crash (PR #47) introduced a regression that caused `lucos_monitoring` to fail all HTTPS fetches with `ssl_not_started`, making every monitored service appear as erroring. The root cause was that `application:ensure_all_started(inets)` does not start the `ssl` OTP application — `inets` only declares `kernel` and `stdlib` as dependencies. Two failed fix attempts pushed untested code to production before the correct single-call fix (`ensure_all_started([ssl, inets])`) was identified and deployed.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-03-13 21:51 | PR #47 (`ensure_all_started(inets)` fix) merged and pipeline 174 triggered |
| 2026-03-13 21:54 | New monitoring container deployed with PR #47 change |
| 2026-03-13 21:54 | All monitored services begin showing `fetch-info: ssl_not_started` — monitoring effectively blind |
| 2026-03-14 ~00:17 | Root cause diagnosed: `inets.app` does not declare `ssl` as a dependency |
| 2026-03-14 ~00:17 | Issue #52 raised; PR #53 opened with `ensure_started(ssl)` + `ensure_started(inets)` fix |
| 2026-03-14 ~00:52 | PR #53 merged |
| 2026-03-14 ~00:54 | Container deployed with PR #53 — immediately begins crash-looping with `{not_started,crypto}` |
| 2026-03-14 ~08:47 | PR #54 opened with corrected fix: `ensure_all_started([ssl, inets])` |
| 2026-03-14 08:52 | PR #54 merged via auto-merge after CI passes |
| 2026-03-14 08:54 | Container deployed and healthy — monitoring restored |

---

## Root Cause

PR #47 was intended to fix `{already_started, X}` crashes that occurred when the production OTP release auto-started applications (via the relx release list) before `fetcher:start/1` ran explicit `application:start/1` calls. The fix replaced those calls with `application:ensure_all_started(inets)`.

However, `inets.app` only declares `kernel` and `stdlib` in its `applications` key — the OTP mechanism that `ensure_all_started` uses to walk the dependency tree. `ssl` appears only in `runtime_dependencies`, which is documentation metadata and is not used by the application controller. So `ensure_all_started(inets)` started `inets` without starting `ssl`, `crypto`, `asn1`, or `public_key`. Every subsequent `httpc:request` call with TLS options returned `{error, ssl_not_started}`.

The `inets.app` file even contains an explicit comment warning about this: "If the 'new' ssl is used then 'crypto' must be started before inets." This is a well-known OTP footgun.

---

## What Was Tried That Didn't Work

### Fix attempt 1: `ensure_started(ssl)` + `ensure_started(inets)` (PR #53)

`application:ensure_started/1` is idempotent but does NOT start an application's dependencies — it only starts the named application itself. Since `ssl` depends on `crypto`, `asn1`, and `public_key`, calling `ensure_started(ssl)` when those dependencies are not yet started causes `{badmatch,{error,{not_started,crypto}}}`. This crashed the monitoring process on every startup, turning a silent data corruption (wrong monitoring state) into a full crash-loop.

This was not caught before merging because the fix was not tested locally with `docker build && docker run`.

---

## Resolution

PR #54 replaced the two-line fix with a single call:

```erlang
{ok, _} = application:ensure_all_started([ssl, inets]),
```

`ensure_all_started/1` accepts a list (OTP 26+) and walks the full dependency tree for each application in order. This starts `crypto`, `asn1`, `public_key`, `ssl`, and `inets` — and is idempotent, returning `{ok, []}` if they are already running (as they are in the production relx release). Service was restored within 2 minutes of the deploy completing.

---

## Contributing Factors

### Factor 1: No local container testing before pushing fixes to production

All three PRs involved in this incident (#47, #53, #54) were pushed and merged without first building and running the container locally. Docker is available in this environment — a `docker build && docker run` would have caught both the original regression and the failed fix attempt within minutes. The standard workflow of "push → wait for CI → check production logs" added hours to each iteration and repeatedly degraded a live service.

### Factor 2: `inets.app` OTP dependency declaration does not include `ssl`

The OTP `inets` application deliberately does not declare `ssl` as a hard dependency because SSL support is optional. This is a documented quirk but easy to overlook — `ssl` appears in `runtime_dependencies` which looks like a dependency list but is not used by `ensure_all_started`.

### Factor 3: Monitoring blind to its own outage

While `lucos_monitoring` was returning `ssl_not_started` for every service, it was still responding to its own `/_info` healthcheck (which doesn't make HTTPS fetches). So the container appeared healthy to Docker and to external health checks, meaning there was no automated alert — only user observation.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Root cause fix: `ensure_all_started([ssl, inets])` | [lucas42/lucos_monitoring#54](https://github.com/lucas42/lucos_monitoring/pull/54) | Done |
| Add local Docker testing rule to agent persona files | (via lucos-issue-manager) | In progress |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
