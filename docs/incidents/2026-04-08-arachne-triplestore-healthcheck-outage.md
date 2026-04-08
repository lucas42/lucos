# Incident: arachne complete outage — wget missing from Fuseki 6.0.0 base image

| Field | Value |
|---|---|
| **Date** | 2026-04-08 |
| **Duration** | ~5 minutes (14:50 UTC to ~14:55 UTC) |
| **Severity** | Complete outage — arachne web, ingestor, and MCP all unavailable |
| **Services affected** | arachne.l42.eu (web, search, MCP, ingestor) |
| **Detected by** | User report (lucas42) |

---

## Summary

A batch of PRs (#258–#268) to `lucos_arachne` included an upgrade of the Fuseki triplestore from 5.5.0 to 6.0.0 (via lucas42/lucos_arachne#268). The new Fuseki 6.0.0 image is built on `openjdk:22-jdk-slim`, which does not include `wget`. The triplestore Docker healthcheck used `wget`, so every healthcheck invocation failed with `executable file not found`. Because the `web`, `ingestor`, and `mcp` containers depend on `triplestore: condition: service_healthy` (added in lucas42/lucos_arachne#256), they remained in `Created` state indefinitely and never started. The triplestore itself was running correctly throughout. Service was restored by manually starting the stuck containers; a permanent fix was shipped in lucas42/lucos_arachne#278.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~14:50 | Batch of PRs (#258–#268) deployed; triplestore container starts with Fuseki 6.0.0 |
| ~14:50 | Triplestore healthcheck begins failing: `exec: "wget": executable file not found in $PATH` |
| ~14:50 | `web`, `ingestor`, `mcp` containers remain in `Created` state — blocked on `service_healthy` |
| ~14:52 | lucas42 reports arachne is completely down |
| ~14:52 | SRE investigation begins — container statuses checked |
| ~14:53 | Root cause identified: triplestore `unhealthy` due to missing `wget`; Fuseki confirmed responsive via bash TCP probe |
| ~14:55 | `docker start lucos_arachne_web lucos_arachne_mcp lucos_arachne_ingestor` — service restored |
| ~15:05 | `web` and `mcp` containers healthy; incident closed |
| ~15:05 | Permanent fix PR lucas42/lucos_arachne#278 opened |
| ~15:30 | PR reviewed, approved, and merged; fix deployed via CI |

---

## Analysis

### Stage 1: Fuseki 6.0.0 base image dropped wget

lucas42/lucos_arachne#268 upgraded the triplestore from Fuseki 5.5.0 to 6.0.0 as part of the PR batch. The previous Fuseki image was pulled directly from Docker Hub (`stain/jena-fuseki` or similar) and included standard Unix utilities. The new build — introduced by lucas42/lucos_arachne#10 — builds Fuseki from source on `openjdk:22-jdk-slim`. The `-slim` variant of OpenJDK Debian images is intentionally minimal and does not include `wget`, `curl`, `nc`, or any HTTP client by default.

The healthcheck in `docker-compose.yml` had always used `wget`:

```yaml
test: ["CMD", "wget", "-qO-", "http://127.0.0.1:3030/$/ping"]
```

This assumption was valid for the old image but silently broke when the base changed.

### Stage 2: service_healthy dependency amplified the blast radius

lucas42/lucos_arachne#256 (merged in the same batch) changed the ingestor's `depends_on` for the triplestore from `condition: service_started` to `condition: service_healthy`. A separate earlier change set the same condition on `web` and `mcp`. This was the correct reliability improvement — it prevents containers starting before the triplestore is ready.

However, it also meant that a triplestore that starts but cannot pass its healthcheck now blocks all downstream containers from starting at all. Previously, with `service_started`, the dependent containers would have started immediately and the broken healthcheck would have caused no cascading effect (just an inaccurate `unhealthy` status).

Two good changes — `service_healthy` dependency and the Dockerfile-based Fuseki build — interacted to produce a complete outage from what would otherwise have been a cosmetic monitoring anomaly.

### Stage 3: The error was masked

The `docker compose up --wait` CI step (from lucas42/lucos_deploy_orb#16) waits for all containers to reach a healthy state before reporting success. In this case, `web`, `ingestor`, and `mcp` never started, so the wait would have timed out — but the containers were left in `Created` state rather than being cleaned up, and CI may have reported the deploy as failed rather than drawing attention to the root cause.

The triplestore itself showed no errors in its logs (`Start Fuseki` was the last message; Fuseki was responding correctly to requests). The failure was purely at the healthcheck wrapper layer, making it non-obvious from logs alone.

---

## What Was Tried That Didn't Work

Initial hypothesis was that PR #268 might have caused a Fuseki crash (reduced JVM heap to 768m, Docker memory to 1G). Triplestore memory was checked: 636.9 MiB / 2 GiB (31%) — healthy. Fuseki logs showed a clean start with no OOM or error messages, ruling out memory pressure as the cause.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add `curl` to triplestore Dockerfile; update healthcheck from `wget` to `curl --fail` | lucas42/lucos_arachne#278 | Done |
| Document `openjdk:22-jdk-slim` healthcheck pattern in SRE memory | — | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
