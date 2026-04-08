# Incident: arachne second outage — ingestor ImportError after live_systems rename

| Field | Value |
|---|---|
| **Date** | 2026-04-08 |
| **Duration** | ~30 minutes (~15:05 UTC to ~15:22 UTC) |
| **Severity** | Complete outage — arachne web unavailable |
| **Services affected** | arachne.l42.eu (web) |
| **Detected by** | User report (lucas42) — monitoring did NOT alert (see Analysis) |

---

## Summary

Approximately 10 minutes after the first arachne outage was resolved (via PR #278), a second outage began. PR #267 (*Cache 3rd party ontology files locally in repo*), part of the same batch deploy, had renamed `systems_to_graphs` to `live_systems` in `ingestor/triplestore.py`. The `ingestor/server.py` file — which handles incoming Loganne webhooks — was not updated to use the new name. The ingestor crash-looped on startup with an `ImportError`, which (via `depends_on: condition: service_healthy`) blocked the `web` container from starting.

Monitoring did not alert because the failure occurred during the deploy silence window; when silence lifted, the monitoring system did not re-evaluate the state and no transition alert was fired. Tracked in lucas42/lucos_monitoring#135.

Service was restored when PR #280 was merged and deployed.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~14:55 | First arachne outage resolved (PR #278 deployed; web/ingestor/mcp manually started) |
| ~15:05 | CI deploy for post-#278 batch PRs begins; monitoring suppression active |
| ~15:05 | Ingestor container replaced with image from PR #267; starts crash-looping: `ImportError: cannot import name 'systems_to_graphs'` |
| ~15:05 | `web` stuck in `Created` — blocked on `ingestor: service_healthy` |
| ~15:05–15:10 | Monitoring suppression window ends; no alert fires (failure occurred during suppression — see Analysis) |
| ~15:10 | lucas42 reports arachne is still down |
| ~15:11 | SRE investigation: ingestor logs show `ImportError: systems_to_graphs` |
| ~15:12 | Root cause identified: PR #267 renamed `systems_to_graphs` → `live_systems` in `triplestore.py`; `server.py` not updated |
| ~15:14 | Fix PR lucas42/lucos_arachne#280 opened (3-line rename in `server.py`) |
| ~15:15 | Code reviewer approves #280 |
| ~15:17 | PR merged; CI deploy begins |
| ~15:22 | Deploy completes; all 6 containers healthy; `/_info` returns 200 |

---

## Analysis

### Stage 1: Incomplete refactor in PR #267

PR #267 (*Cache 3rd party ontology files locally in repo*) split the old `systems_to_graphs` dict in `triplestore.py` into two dicts: `live_systems` (lucos services, fetched over HTTP) and `ontology_cache` (3rd party ontologies, read from local files). `ingest.py` was correctly updated in the same PR to use `live_systems`. However, `server.py` — the webhook handler — was not updated. It retained the import `from triplestore import systems_to_graphs, ...`, which raised an `ImportError` at startup.

The ingestor has two separate entry points:
- `ingest.py` — bulk ingestion on startup and via cron
- `server.py` — the HTTP webhook handler for real-time Loganne events

Both import from `triplestore.py`, but PR #267 only updated `ingest.py`. `server.py` is a less frequently touched file and was overlooked.

### Stage 2: service_healthy dependency amplified blast radius (again)

As with the first outage, the `web` container's `depends_on: ingestor: condition: service_healthy` meant that the ingestor crash-loop blocked the web container entirely. The ingestor started and immediately crashed; Docker's `restart: always` kept retrying but it never reached a healthy state.

### Stage 3: Monitoring did not alert

The failure began during the monitoring suppression window triggered by the deploy. The monitoring system only fires alerts on state transitions (healthy → unhealthy). When suppression ended, the transition detector compared the current state to the pre-suppression state — but because the transition happened *during* suppression, it was invisible to the detector. From the monitor's perspective, nothing had changed.

This is a structural gap: monitoring suppression effectively hides failures that begin mid-deploy and persist after the window closes. Tracked as lucas42/lucos_monitoring#135.

---

## What Was Tried That Didn't Work

No false leads this time — the `ImportError` in the logs immediately identified the affected file and function name. Root cause was confirmed in under two minutes.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix `server.py` to use `live_systems` instead of `systems_to_graphs` | lucas42/lucos_arachne#280 | Done |
| Fix monitoring: alert when service is unhealthy at end of suppression window | lucas42/lucos_monitoring#135 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
