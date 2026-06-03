# Incident: lucos_backups file-descriptor exhaustion (EMFILE) blinds backup tracking

| Field | Value |
|---|---|
| **Date** | 2026-06-03 |
| **Duration** | ~9 hours tracking blind (23:07 UTC 2026-06-02 to 08:14 UTC 2026-06-03); alerting from 01:07 UTC |
| **Severity** | Partial degradation / Data risk |
| **Services affected** | lucos_backups (backup tracking; scheduled `create-backups` next in line to fail) |
| **Detected by** | SRE ops check (Monitoring API), the morning after alerting began |

---

## Summary

The `lucos_backups` web/job process (`python -m server`) slowly leaked file descriptors over ~5 days of uptime until it hit its 1024 soft limit. Once exhausted, every outbound connection failed with `[Errno 24] No file descriptors available` (EMFILE), so the hourly `tracking` and `config` jobs could no longer reach the backup hosts, GitHub, configy, or schedule-tracker — leaving backup tracking blind for ~9 hours. The shallow Docker healthcheck stayed green the whole time. Service was restored by restarting the container (which reset the fd count) and re-running the jobs end-to-end; root cause is a connection leak in the tracking code path, filed as `lucas42/lucos_backups#291`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-06-02 23:03 / 23:07 | Last successful `config` / `tracking` runs — fd headroom runs out shortly after |
| 2026-06-02 23:07 onward | Every hourly `tracking`/`config` cron run fails with EMFILE; `data-age` begins to stale |
| 2026-06-03 01:07 | First `monitoringAlert`: `data-age` failing |
| 2026-06-03 02:07 | 3 failing checks: `config`, `data-age`, `tracking` |
| 2026-06-03 ~08:05 | SRE ops check picks up `lucos_backups` failing (1 of 52 systems) |
| 2026-06-03 08:08 | Diagnosed EMFILE: PID 22 holding 1022/1024 fds, 1019 sockets; 16 orphaned `ssh-agent` processes |
| 2026-06-03 08:13 | `docker restart lucos_backups` — fd count resets 1022 → 6, ssh-agents 16 → 1 |
| 2026-06-03 08:14 | Triggered `refresh-config` + `refresh-tracking` end-to-end; both succeeded |
| 2026-06-03 ~08:16 | Monitoring re-polls: 52/52 healthy, `lucos_backups` green |

---

## Analysis

### Root cause: connection leak in the tracking path

`utils/tracking.py::fetchAllInfo()` (the hourly `refresh-tracking` job) constructs a `Host` per backup host via `Host.getAll()`. Each `Host.__init__` opens a `fabric.Connection` (and a **second** gateway `Connection` for gatewayed hosts such as aurora-via-xwing), but the function **never calls `host.closeConnection()`**. The `create-backups` and `prune-backups` scripts do close their connections; the tracking path does not. So each hourly run leaks roughly one paramiko transport socket per host (plus agent-forward channels and gateway connections). Over ~120 hourly runs (~5 days) this accumulated to the 1019 sockets observed, hitting the process's 1024 soft `nofile` limit.

A sub-factor: `Host.closeConnection()` only closes `self.connection`, never the separately-constructed `gateway` connection — so gateway connections leak even from the scripts that *do* close their main connection.

### Why it took 5 days, then failed all at once

A file-descriptor leak is a slow ramp with a hard cliff: nothing visibly wrong until the table is full, then every operation needing a new fd fails simultaneously. The failure landed at ~23:07, so the first user-meaningful symptom (`data-age` going stale past its 2-hour threshold) didn't alert until 01:07 — overnight, hence the ~9-hour blind window before the ops check caught it.

### Detection gap: shallow healthcheck masked the failure

The Docker healthcheck for `lucos_backups` does not exercise any job path that opens a socket, so the container reported `Healthy` throughout the EMFILE window. This is the same class of "green healthcheck, broken job" gap seen before on this service — Docker liveness is not job-liveness. Monitoring's schedule-tracker-backed `tracking`/`config` checks and the `data-age` check are what actually caught it (correctly), which is the system working as designed; the healthcheck simply isn't a useful signal here.

### Secondary leak: orphaned ssh-agents from cron

Independently, `scripts/init-agent.sh` spawns `ssh-agent -s` whenever `SSH_AUTH_SOCK` is unset. `startup.sh` sources it once (fine), but each `create-backups` (2×/day) and `prune-backups` (1×/day) cron entry re-sources it in a fresh shell that doesn't inherit `SSH_AUTH_SOCK`, spawning a new agent that is never killed when the cron shell exits. Exactly 16 orphans were observed (15 from 5 days × 3/day + 1 from startup). This did not cause the EMFILE (those fds live in the agent processes, not the server), but it is an unbounded process/fd leak in its own right and is fixed in the same issue.

---

## What Was Tried That Didn't Work

- **First trigger attempt hit the wrong port.** `curl` to `127.0.0.1:8000` returned connection-refused (000); the server listens on `$PORT` (8027), not 8000. Re-issued against 8027 and the jobs succeeded. Minor, but noted so the next responder reaches for `$PORT` directly.
- Host-level `/proc/<pid>/fd` inspection needed root (no passwordless sudo for the agent SSH user); `docker exec … cat /proc/<pid>/limits` and `ls /proc/<pid>/fd` from *inside* the container worked without elevation and gave the same evidence.

Otherwise the diagnosis-to-fix path was first-attempt: the `[Errno 24]` log lines named the failure mode immediately, and the restart restored service cleanly.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Close fabric connections in `fetchAllInfo()` (use `try/finally`); also close the gateway connection in `closeConnection()` | lucas42/lucos_backups#291 | Open |
| Stop cron jobs leaking `ssh-agent` processes (shared agent or `trap … EXIT` kill) | lucas42/lucos_backups#291 | Open |
| Consider a regression guard asserting the server's open-fd count is stable across N tracking runs | lucas42/lucos_backups#291 | Open (proposed) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

(The leak involves SSH connections that use a private key held in `ssh-agent`, but no key material or credential was exposed or is reproduced here.)
