# Incident: avalon router crash-loop — stale nginx configs referencing deleted SSL certs

| Field | Value |
|---|---|
| **Date** | 2026-04-02 |
| **Duration** | ~29 minutes (00:21 UTC to 00:50 UTC) |
| **Severity** | Complete outage — all HTTP services on avalon unreachable |
| **Services affected** | All services proxied through the avalon router: repos.l42.eu, monitoring.l42.eu, configy.l42.eu, contacts.l42.eu, photos.l42.eu, eolas.l42.eu, backups.l42.eu, loganne.l42.eu, and ~20 others |
| **Detected by** | User report (repos.l42.eu unreachable) |

---

## Summary

A batch of ~46 PR merges included a `lucos_configy` deploy, which triggered a restart of the `lucos_router` container on avalon. On restart, nginx attempted to load all configs from the persistent `generatedconfig` Docker volume — including 5 stale configs for domains that had been removed from service. Those stale configs referenced SSL certificates that had been cleaned up from the `letsencrypt` volume (see lucas42/lucos_router#38). Because nginx fails hard on missing certificate references, every restart attempt crashed, putting the router in a crash-loop and taking all avalon services offline. Service was restored by manually removing the 5 stale configs from the volume and restarting the container.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~00:20 | ~46 PRs merged in a batch; configy deploy triggered among others |
| ~00:21 | `lucos_router` deploy job starts; container restarts as part of normal deploy |
| ~00:21 | nginx attempts to load all configs from `generatedconfig` volume; fails on `avalon.file-sync.l42.eu.conf` (missing cert) |
| ~00:21 | Router enters crash-loop (`restart: always`); all avalon HTTP services become unreachable |
| ~00:22 | 8 other services' deploy jobs fail — containers themselves are healthy but `docker compose up --wait` times out or deploy CI infrastructure is unreachable |
| ~00:40 | SRE investigation begins following user report |
| ~00:43 | Root cause identified: 5 stale configs in `generatedconfig` volume reference deleted certs |
| ~00:47 | Stale configs removed from volume: `avalon.file-sync.l42.eu.conf`, `googlecontactsync.l42.eu.conf`, `speak.l42.eu.conf`, `valen.file-sync.l42.eu.conf`, `valen.s.l42.eu.conf` |
| ~00:48 | Router restarted; nginx starts cleanly; `update-domains.sh` runs and regenerates all configs |
| ~00:50 | Router healthy; all avalon services reachable |
| ~00:51 | lucos_time eolas cache check still red (fetch had failed at container startup during outage); container restarted |
| ~00:52 | All monitoring checks green |
| ~01:10–01:30 | 9 failed CI pipelines identified and re-run via CircleCI API; all pass |
| ~01:35 | Monitoring fully clear |

---

## Analysis

### Stage 1: Cert cleanup left stale nginx configs in the persistent volume

lucas42/lucos_router#38 (filed 2026-03-23) identified several expired Let's Encrypt cert directories for defunct domains in the letsencrypt volume. Those cert directories were subsequently removed. However, the corresponding nginx `.conf` files in the `generatedconfig` volume were not removed — `update-domains.sh` regenerates configs for active domains but never purges configs for domains that no longer exist in configy.

This created a latent defect: 5 configs in the persistent `generatedconfig` volume referenced certificates that no longer existed on disk.

### Stage 2: configy deploy triggered a router restart

A batch of ~46 PR merges included a `lucos_configy` deploy. Because `lucos_router` depends on configy, this triggered a redeploy of the router. Under normal circumstances this is routine — the router restarts, `update-domains.sh` runs, certs are renewed as needed.

### Stage 3: nginx fails hard on missing cert references at startup

When the router container starts, `startup.sh` launches nginx in the background before `update-domains.sh` runs. nginx loads all `.conf` files from the `generatedconfig` volume at startup. The first file it encountered with a missing cert (`avalon.file-sync.l42.eu.conf`) caused nginx to exit immediately with:

```
nginx: [emerg] cannot load certificate "/etc/letsencrypt/live/avalon.file-sync.l42.eu/fullchain.pem": BIO_new_file() failed (SSL: error:80000002:system library::No such file or directory)
```

With `restart: always` set, Docker immediately restarted the container, which hit the same error. This crash-loop continued for ~29 minutes until the stale configs were removed.

### Stage 4: Bootstrap dependency amplified the impact

`update-domains.sh` fetches the domain list from `https://configy.l42.eu/hosts/http` — a URL that routes through the router itself. Had the stale configs not prevented nginx from starting, the router could have bootstrapped using its existing `generatedconfig` (which included a valid `configy.l42.eu.conf`). The crash-loop prevented this entirely, leaving no self-healing path.

### Stage 5: Cascading CI failures

During the outage window, 8 other service deploy pipelines ran and failed — some because `docker compose up --wait` could not complete (the high load from 46 simultaneous deploys caused healthcheck timeouts, as in the 2026-03-20 load spike incident), others potentially because Loganne was unreachable for the deploy notification step. All 8 services remained running on their previously-deployed versions; the CI failures were cosmetic rather than functional.

---

## What Was Tried That Didn't Work

Everything tried worked on the first attempt once the root cause was identified. Diagnosis was straightforward: `docker logs lucos_router` immediately showed the nginx cert error, and cross-referencing the `generatedconfig` volume contents with the `letsencrypt` volume identified all 5 problematic files within minutes.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix `update-domains.sh` to purge stale configs for domains no longer in configy | lucas42/lucos_router#58 | Open |
| Make deploy `Notify Loganne` step non-blocking so loganne outages don't fail CI | lucas42/lucos_deploy_orb#42 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
