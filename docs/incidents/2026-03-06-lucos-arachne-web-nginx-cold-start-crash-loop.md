# Incident: lucos_arachne_web crash-loops on cold start — nginx upstream DNS resolution failure

| Field | Value |
|---|---|
| **Date** | 2026-03-06 |
| **Duration** | ~7 minutes (12:31 UTC issue raised to 12:38 UTC PR merged). Service was likely unavailable from the time of the host reboot earlier in the day until recovery. |
| **Severity** | Complete service outage for lucos_arachne_web |
| **Services affected** | lucos_arachne (arachne.l42.eu) — web frontend component |
| **Detected by** | lucos-system-administrator during post-reboot health check |

Source issue: [lucas42/lucos_arachne#60](https://github.com/lucas42/lucos_arachne/issues/60)

---

## Summary

After the avalon host was rebooted on 2026-03-06, `lucos_arachne_web` crash-looped because nginx failed to start when the `search` upstream container was not yet resolvable on the Docker network. nginx resolves all upstream hostnames at config load time by default; if any upstream is not resolvable, nginx exits with a fatal error. The fix was to use variable-based upstream hostnames (`set $upstream_host "search";`) combined with Docker's embedded DNS resolver, which defers DNS resolution to request time rather than startup.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~12:24 | avalon host rebooted. Containers begin starting in arbitrary order. |
| ~12:24–12:31 | `lucos_arachne_web` repeatedly crash-loops: nginx exits with `host not found in upstream "search"`. Other containers start successfully. |
| 12:31 | lucas42/lucos_arachne#60 raised by lucos-system-administrator with diagnosis and three suggested fixes. |
| 12:35 | lucos-developer picks up the issue, selects option 1 (variable-based upstream). |
| 12:38 | lucas42/lucos_arachne#61 merged. `lucos_arachne_web` starts successfully without race condition. Service restored. |

---

## Root Cause

nginx resolves all proxy upstream hostnames at configuration load time (startup). If any upstream hostname cannot be resolved via DNS at that moment — which is likely during a cold host boot, before Docker's embedded DNS has registered all container names — nginx exits fatally.

This had not previously been observed because containers were typically started one at a time or the startup order happened to work out. A full cold-start reboot exposed the race condition.

The fix uses a variable in the `proxy_pass` directive, which changes nginx's behaviour to defer DNS resolution to request time:

```nginx
resolver 127.0.0.11 valid=30s;
set $upstream_host "search";
proxy_pass http://$upstream_host:port;
```

This was applied to all four upstreams (`search`, `explore`, `ingestor`, `triplestore`) to prevent recurrence on any of them.

---

## What Was Tried That Didn't Work

Timeline details not available; no failed attempts recorded in the issue discussion.

---

## Resolution

lucas42/lucos_arachne#61 updated `lucos_arachne_web`'s nginx config to use variable-based upstream hostnames with Docker's embedded DNS resolver (`127.0.0.11`). All four upstream services were updated.

---

## Contributing Factors

### Factor 1: nginx's default startup behaviour fails on unresolvable upstreams

This is documented nginx behaviour but a common gotcha in Docker Compose environments where container startup order is not guaranteed. The service had been running without incident prior to the reboot because the containers had always been started incrementally. The reboot revealed a latent race condition.

### Factor 2: No `restart: on-failure` or health-check-based `depends_on`

The `depends_on` directive only waits for the upstream container to start, not for it to be healthy and registered in Docker DNS. A health-check-based `depends_on` would have delayed `lucos_arachne_web` startup until its upstreams were ready, but the variable-based upstream approach is more robust as it handles the case where an upstream becomes temporarily unavailable after startup too.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix applied via PR #61 (variable-based nginx upstreams) | [lucas42/lucos_arachne#61](https://github.com/lucas42/lucos_arachne/pull/61) | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
