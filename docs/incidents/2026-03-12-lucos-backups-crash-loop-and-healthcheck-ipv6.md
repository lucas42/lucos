# Incident: lucos_backups crash loop and persistent deploy failures

| Field | Value |
|---|---|
| **Date** | 2026-03-12 |
| **Duration** | ~83 minutes — 19:27 UTC to 20:50 UTC (service outage); deploy CI red since lucas42/lucos_backups#47 merged (2024-era, chronic) |
| **Severity** | Complete outage (backups.l42.eu unreachable) |
| **Services affected** | lucos_backups |
| **Detected by** | User report / team-lead investigation |

---

## Summary

lucas42/lucos_backups#55 refactored lucos_backups to use the `lucos-loganne-pythonclient` and `lucos-schedule-tracker-pythonclient` PyPI packages in place of hand-rolled local utilities. Unlike the previous code, both PyPI clients read the `SYSTEM` environment variable at import time and call `sys.exit()` if it's not set. Since `SYSTEM` was never in the `docker-compose.yml` environment passthrough (the old code hardcoded the system name), every container start crashed immediately, causing an 83-minute outage. During investigation, a second pre-existing bug was discovered: the Docker healthcheck had used `localhost` since it was added in lucas42/lucos_backups#47, which Alpine resolves to `::1` (IPv6) rather than `127.0.0.1`. The server only binds IPv4, so the healthcheck had been failing on every start since lucas42/lucos_backups#47 — causing every deploy since then to be reported as failed in CircleCI, even when the service was actually running.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~19:27 | lucas42/lucos_backups#55 (PyPI client migration) merges to main, triggering a deploy |
| 19:27–19:58 | Deploy pipelines 131, 133, 137 all fail: container crashes on startup with `SYSTEM environment variable not set` |
| ~20:13 | User reports service down; team-lead asks SRE to investigate |
| ~20:20 | SRE confirms 502 on `backups.l42.eu/_info` via monitoring API; container logs show crash loop |
| ~20:25 | Root cause identified: `loganne.py` (PyPI client) calls `sys.exit()` if `SYSTEM` not in environment |
| ~20:47 | lucas42/lucos_backups#56 (add `SYSTEM`, `ENVIRONMENT`, `APP_ORIGIN` to docker-compose environment passthrough) merged |
| ~20:50 | New container starts successfully; service restored; logs show `Server started on port 8027` |
| ~20:51 | Pipeline 139 (deploy of the old image, already in flight) reports as failed — false alarm |
| ~20:53 | SRE triggers fresh pipeline 140 on main to clear the red CI check |
| ~20:58 | Pipeline 140 also fails: healthcheck returns `Connection refused` after 53 seconds |
| ~20:59 | Root cause of CI failure identified: healthcheck uses `localhost`, resolves to `::1` on Alpine |
| ~21:00 | lucas42/lucos_backups#58 (change healthcheck to `127.0.0.1`) opened |
| ~21:01 | lucas42/lucos_backups#58 CI (branch build) passes |
| ~21:02 | lucas42/lucos_backups#58 merged |
| ~21:04 | Deploy pipeline succeeds; CircleCI check goes green; all 10 monitoring checks pass |

---

## Root Cause

### Primary: Missing `SYSTEM` env var passthrough (lucas42/lucos_backups#55 regression)

The `lucos-loganne-pythonclient` package (v1.0.4) contains this at the top of `loganne.py`:

```python
try:
    SYSTEM = os.environ["SYSTEM"]
except KeyError:
    sys.exit("\033[91mSYSTEM environment variable not set\033[0m")
```

`lucos-schedule-tracker-pythonclient` has identical logic for `SYSTEM`. Both modules are imported at startup, so if `SYSTEM` is absent the process exits immediately — before the HTTP server starts, before the healthcheck can pass.

The old hand-rolled `utils/loganne.py` hardcoded `source = "lucos_backups"` and required no env vars. When lucas42/lucos_backups#55 replaced it with the PyPI client, it didn't add `SYSTEM` to the `docker-compose.yml` `environment:` passthrough. lucos_creds provides `SYSTEM` in the `.env` file, but Docker Compose only passes named variables through to the container — unlisted vars are silently dropped.

### Secondary (pre-existing): Healthcheck `localhost` resolves to IPv6 on Alpine

The `docker-compose.yml` healthcheck (added in lucas42/lucos_backups#47) used:

```yaml
test: ["CMD", "wget", "-qO-", "http://localhost:${PORT}/_info"]
```

With `network_mode: host` on Alpine Linux, musl libc resolves `localhost` to `::1` (IPv6) rather than `127.0.0.1`. The Python server binds to `0.0.0.0` (IPv4 only), so the healthcheck always got `Connection refused`. Docker therefore kept the container in `unhealthy` state permanently, and `docker compose up --wait` (used by the deploy orb) always timed out and reported failure — even when the service was serving traffic correctly on IPv4.

This meant every deploy since lucas42/lucos_backups#47 had been marked failed in CircleCI. The service stayed up because `restart: always` kept the container running and the nginx reverse proxy only uses IPv4.

---

## What Was Tried That Didn't Work

- **Triggered a manual pipeline on main (pipeline 140) to clear the red CI check.** This also failed — at that point the `SYSTEM` issue was resolved (PR #56 was deployed) but the healthcheck IPv6 bug then caused the `docker compose up --wait` to time out. This is what revealed the second bug.
- **Initially assumed the service had recovered when container logs showed `Server started on port 8027`.** It had recovered for external traffic, but Docker still reported the container as `unhealthy` due to the IPv6 healthcheck issue — which is why the deploy kept failing.

---

## Resolution

Two PRs:

- **lucas42/lucos_backups#56**: Added `SYSTEM`, `ENVIRONMENT`, and `APP_ORIGIN` to the `environment:` passthrough in `docker-compose.yml`. These are the standard lucos vars that lucos_creds provides to every service. Restored service after 83 minutes of downtime.
- **lucas42/lucos_backups#58**: Changed the Docker healthcheck from `http://localhost:${PORT}/_info` to `http://127.0.0.1:${PORT}/_info`. Resolved the persistent deploy failure and cleared the red CircleCI check on the monitoring dashboard.

---

## Contributing Factors

### PyPI clients have import-time env var requirements not present in local utils

The local utilities had no env var requirements — they hardcoded values or used minimal config. The PyPI clients are more correct (reading config from the environment) but impose requirements that the migrating service may not yet satisfy. There was no checklist or documentation to prompt checking env var requirements when switching to a PyPI client.

### Healthcheck bug masked by service appearing externally healthy

Because `restart: always` kept the container running and the reverse proxy only cares about IPv4 reachability, there was no user-visible symptom of the healthcheck failure. The only signal was the red CircleCI check — which, because deploys had been failing since lucas42/lucos_backups#47, may have been normalised or overlooked.

### Stale queued pipeline caused confusion during recovery

When PR #56 merged, there was already a deploy pipeline in flight using the old (broken) image. That pipeline reported failure. At the time it wasn't clear whether this was a stale pipeline or a sign that PR #56 hadn't fixed the problem, which prompted an unnecessary investigation loop.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add `SYSTEM`, `ENVIRONMENT`, `APP_ORIGIN` to docker-compose environment passthrough | lucas42/lucos_backups#56 | Done |
| Fix healthcheck to use `127.0.0.1` instead of `localhost` | lucas42/lucos_backups#58 | Done |
| Document that lucos PyPI clients require `SYSTEM` and `LOGANNE_ENDPOINT` at import time | — | Open — lucos-issue-manager to file |
| Consider adding a lucos_repos convention check: services using lucos PyPI clients must have the required env vars in their docker-compose | — | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
