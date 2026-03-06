# Incident: lucos_repos down — x509 TLS certificate verification failure

**Date:** 2026-03-06
**Duration:** ~39 minutes (00:18 UTC to 01:38 UTC on 2026-03-06, based on earliest/latest log entries)
**Severity:** Complete service outage (repos.l42.eu returning 502 Bad Gateway)
**Services affected:** lucos_repos (repos.l42.eu)
**Detected by:** lucos-site-reliability ops check

Source issue: [lucas42/lucos_repos#39](https://github.com/lucas42/lucos_repos/issues/39)

---

## Summary

The `lucos_repos` container was fully unable to contact the GitHub API, returning 502 on every request. Every log line showed `x509: certificate signed by unknown authority` when trying to reach `api.github.com`. The root cause was that the `debian:trixie-slim` base image used in the runtime stage of the Dockerfile does not include a CA certificate bundle — so the Go binary had no trusted CA anchors. The fix was to add `ca-certificates` to the runtime image and redeploy.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 00:18 | Earliest log entry from 2026-03-06 showing the TLS failure. Service likely down from this point. |
| 00:59 | Issue #39 raised by lucos-site-reliability with root cause diagnosis. |
| 01:36 | lucos-developer responds with diagnosis confirmation and fix plan. |
| 01:38 | PR #40 merged: `ca-certificates` added to runtime Dockerfile stage. Container rebuilt and deployed. Service restored. |

---

## Root Cause

The Dockerfile for `lucos_repos` uses a multi-stage build. The runtime stage was based on `debian:trixie-slim`, which is a minimal image that does not include the `ca-certificates` package. Without this package, the Go binary's TLS client has no trusted CA bundle and cannot verify any TLS certificate, including those used by `api.github.com`.

This was not caught earlier because the service was newly rebuilt from Go (PR #36, deployed 2026-03-05 at 23:10 UTC) and the TLS verification failure manifested on the very next startup — within hours of initial deployment.

---

## What Was Tried That Didn't Work

A container restart was attempted (noted in the issue body) and did not resolve the issue — the error recurred immediately on startup, confirming it was a structural image problem rather than a transient state issue.

---

## Resolution

PR #40 added `apt-get install -y ca-certificates` to the runtime stage of the Dockerfile. A rebuild and redeploy via CI fixed the outage without any manual server-side changes.

---

## Contributing Factors

### Factor 1: Minimal base image lacks CA certificates by default

`debian:trixie-slim` intentionally omits non-essential packages to keep image size small. This means `ca-certificates` must be explicitly installed in any container whose application makes outbound HTTPS connections. This is a well-known gotcha but easy to miss when scaffolding a new service.

### Factor 2: No integration test covering external HTTPS calls

The service had no test that exercised the GitHub API call path, so the missing CA bundle was not caught in CI before the image was deployed.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix applied via PR #40 (add `ca-certificates` to runtime image) | [lucas42/lucos_repos#40](https://github.com/lucas42/lucos_repos/pull/40) | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
