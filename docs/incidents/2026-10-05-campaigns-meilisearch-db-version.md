# Incident: lucos_campaigns down after a Meilisearch patch bump

| Field | Value |
|---|---|
| **Date** | 2026-10-05 |
| **Duration** | ~2h30m (21:23 UTC to 23:53 UTC) |
| **Severity** | Complete outage |
| **Services affected** | lucos_campaigns (campaigns.l42.eu) |
| **Tracking issue** | lucas42/lucos_campaigns#66 |
| **Detected by** | Monitoring alerts at 21:30/21:33 UTC; picked up at ~23:39 UTC by the sysadmin and SRE ops checks independently |

---

## Summary

A Dependabot patch bump of Meilisearch from v1.54.1 to v1.54.3 was auto-merged into lucos_campaigns. On deploy, the search container refused to open its existing on-disk index ("database version incompatible") and crash-looped. The Kanka app container crash-looped behind it, because its start-up search import couldn't reach the search host. campaigns.l42.eu was down until a one-line config change (`MEILI_UPGRADE_DB=true`) let Meilisearch migrate the index in place. Service was restored at 23:53 UTC, after about 2½ hours.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 21:17 | lucas42/lucos_campaigns#65 (Dependabot `minor-and-patch` group: Meilisearch v1.54.1 → v1.54.3, oauth2-proxy v7.15.4 → v7.15.5) merges |
| 21:18–21:21 | CI `test-gate`, `test-migration`, `test-info` and `lucos/build` all pass |
| 21:23 | Deploy recreates `lucos_campaigns_search` and `lucos_campaigns_app`. Search exits on `Your database version (1.54.1) is incompatible with your current engine version (1.54.3)` and starts restarting about once a minute |
| 21:25 | `lucos/deploy-avalon` fails: `dependency failed to start: container lucos_campaigns_search is unhealthy` |
| 21:30 | Monitoring alert: `lucos_docker_health` avalon, crash-looping `lucos_campaigns_app`, `lucos_campaigns_search` |
| 21:33 | Monitoring alert: `lucos_campaigns` `fetch-info` (timeout) and `circleci` |
| 23:39 | SRE ops check picks up both alerts. Logs give the cause directly |
| 23:40 | lucos-system-administrator's ops check independently files lucas42/lucos_campaigns#66 |
| 23:40 | Fix reproduced locally: index created on v1.54.1 opens on the pinned v1.54.3 digest with `MEILI_UPGRADE_DB=true`, data intact |
| ~23:41 | Hotfix lucas42/lucos_campaigns#67 opened |
| 23:47:55 | lucas42/lucos_campaigns#67 merged (closes lucas42/lucos_campaigns#66) |
| 23:53:03 | New search container starts with the flag set and logs `Task queue upgraded`. Index intact: 833 documents |
| 23:53:24 | lucos_campaigns v1.0.39 deployed; search and app both `healthy`, restart count 0 |
| 23:53:37 | Monitoring recovery: `lucos_docker_health` |
| 23:54:21 | Monitoring recovery: `lucos_campaigns`. Estate 56/56 healthy. Verified by hand: `/_info` 200 with every check ok, a live search query against the `entities` index returns hits, and no Laravel `production.ERROR` since the new search container started |

---

## Analysis

### Meilisearch treats every engine version bump, patch releases included, as a database migration

Meilisearch stamps its on-disk database with the engine version that wrote it. Unless told to upgrade, it refuses to start when the stamp doesn't match the running engine, even for a patch release. The compose file pinned the image by version and digest but didn't set `MEILI_UPGRADE_DB`. So any Meilisearch bump that reached production was guaranteed to take search down, and the app with it.

The fix sets `MEILI_UPGRADE_DB=true`, so Meilisearch migrates the index in place on start-up. This is safe here because the index volume is `recreate_effort: automatic` / `skip_backup: true` in configy: it is rebuilt from MariaDB, so a migration that went wrong would cost a re-index, not data.

### CI could not catch it

`test-migration` and `test-info` both start from fresh, empty volumes. With no existing database to open there is no version stamp to mismatch, so CI was green on exactly the change that broke production. The only stage that runs against real state is the deploy itself, and it did fail (`dependency failed to start`). By then, though, the containers had already been recreated on the new image and were left crash-looping. The deploy has no rollback.

### The app failure is a dependent, not a second fault

`lucos_campaigns_app` has `depends_on: lucos_campaigns_search: condition: service_healthy`, but `restart: always` restarts it independently of that condition. Each start runs Kanka's Meilisearch import, which fails with `Could not resolve host: lucos_campaigns_search`. Most likely the search container is between restarts at that moment and so has no DNS entry on the compose network, though this hasn't been verified directly. The app's 266 restarts by 23:38 are a symptom of search being down. They need no fix of their own.

### Two hours from alert to action

Both alerts fired within ten minutes of the deploy. Nothing acted on them until the next ops check, about two hours later. This is the same alert→action gap discussed in lucas42/lucos#290 (still open). It's recorded here as one more data point, not as a new finding.

---

## What Was Tried That Didn't Work

Nothing. A container restart was considered and rejected without being tried: the failure is deterministic on start-up, so a restart would only add one more iteration to the crash loop.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Set `MEILI_UPGRADE_DB=true` on lucos_campaigns_search (closes lucas42/lucos_campaigns#66) | lucas42/lucos_campaigns#67 | Done |

No further follow-ups proposed, deliberately:

- **CI test against a previous-version volume.** This would catch the class at build time, but the fix already makes this specific class a non-event for Meilisearch. A general "upgrade from previous image" test would need per-stateful-image fixtures across the estate, which is a large maintenance cost for an outage mode we have now hit once.
- **Deploy rollback in the `lucos/deploy` orb.** Rollback would have shortened this outage, but it's an orb-wide design change, and this incident alone doesn't justify it.

TBD: fold in team responses.

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
