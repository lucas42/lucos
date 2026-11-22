# Incident: A development copy of lucos_backups pruned the production backup hosts

| Field | Value |
|---|---|
| **Date** | 2026-11-20 to 2026-11-22 |
| **Duration** | About 2 days of exposure (dev copy active from ~16:48 UTC on 11-20 until lucas42 stopped it on 11-22). `prune-backups` red from 04:51 to 18:11 UTC on 11-22 (13h20m), cleared by an ad-hoc rerun |
| **Severity** | Data risk / Security |
| **Services affected** | lucos_backups (the backup copies on xwing, aurora, salvare and avalon) |
| **Detected by** | SRE ops check (the `prune-backups` monitoring check) |

---

## Summary

A restart of OrbStack on lucas42's laptop brought back development copies of his lucOS systems, which have `restart: always` set. The development `lucos_backups` holds a working SSH key, which lucas42 reports looks the same as the production key (lucas42/lucos_backups#442). It takes its host list from configy like any other copy, so for about two days it ran its scheduled jobs against the **production** backup hosts. Its `prune-backups` ran at 04:49 UTC alongside the real one on 11-21 and 11-22, and deleted the same retention cohorts at the same time. Both runs failed, which turned the `prune-backups` check red. The leading hypothesis was confirmed by TBD (pending the laptop's `docker logs` and the 2026-11-23 run). The retention ladder is intact on xwing and salvare (verified). Aurora is still to be verified (TBD). The dev copy reported only to local services, so production monitoring never saw its runs, only the damage they did to the real ones. Tracked from lucas42/lucos_backups#365.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-15 → | Not part of this incident, but found during it: lucos_router's nightly schedule-tracker report has been rejected (HTTP 400) since the v2 migration (lucas42/lucos_router#114) |
| 2026-07-04, 07-05, 08-20, 08-27, 08-28 | Earlier `prune-backups` failures of the same shape, recorded on lucas42/lucos_backups#365 and diagnosed at the time as the job racing itself. In hindsight these may be earlier sessions of the same dev copy. The router log only reaches back to 10-29, so this can't be shown |
| 2026-11-20 16:48:57 | First requests from dev-looking clients reach production from the home network address: a Java client (16:48:57), `lucos_creds_configy_sync` (16:48:59), `lucos_dns_sync` (17:00:01). This is consistent with an OrbStack restart at about this time |
| 2026-11-20 17:03:00 | First `lucos_backups` user-agent request from the home network address, configy `/hosts` and `/volumes`, at exactly the job's hourly `refresh-config` slot. There were none in the router log from 11-15 to 11-19 |
| 2026-11-21 04:49 | Both `prune-backups` runs collide. The production run's `rm -rf` of aurora's expired photos snapshot loses its own top-level directory mid-run (1,484 ENOENT lines), and xwing's file listing sees files vanish. The run reports `Prune failed for: aurora.lan/lucos_photos_photos, xwing.s.l42.eu (host unreachable)` |
| 2026-11-22 04:49–04:50 | Same again (448 ENOENT lines on aurora). xwing is skipped by the production run, yet xwing's directories are modified at 04:50:30 and 04:50:33, after the production run had finished at 04:50:26.94 |
| 2026-11-22 04:51 | `prune-backups` turns red (two consecutive failed runs) |
| 2026-11-22 ~16:30 | SRE ops check finds the second deleter. One probe was wrong and wrongly ruled out the development environment (see "What Was Tried That Didn't Work") |
| 2026-11-22 ~17:30 | lucas42 suggests the OrbStack dev copies. SRE confirms the dev environment holds a real key and matches the router-log onset |
| 2026-11-22 (evening) | lucas42 stops all dev containers on the laptop. Retention ladder verified intact on xwing and salvare |
| 2026-11-22 17:03 → 18:03 | Last dev-copy `lucos_backups` request from the home network at 17:03:00; none at the 18:03 slot |
| 2026-11-22 ~18:05 | Ad-hoc `prune-backups` rerun: clean, 1 instance pruned (on aurora) |
| 2026-11-22 18:11:39 | `prune-backups` healthy; dashboard 56/56 |
| 2026-11-23 04:49 | First scheduled `prune-backups` run with the dev copy gone; the test of the hypothesis. TBD pending result |

---

## Analysis

### A development copy is a production actor

Three things combined to make a development container act on production. The second is the one that matters.

1. **A development copy runs the same logic as production, by design.** `src/utils/config.py` fetches hosts and volumes from `https://configy.l42.eu`, and the crontab is the same in every environment: `create-backups` at 03:25 and 15:25, `prune-backups` at 04:49, and `refresh-tracking` hourly. That's deliberate. A development version whose core logic branches on `ENVIRONMENT` wouldn't be worth having (see Follow-up Actions). So what keeps a dev copy off the production hosts is not its configuration but its lack of a credential the hosts accept.
2. **The development store holds a working key, and that is the defect.** `creds-shape lucos_backups development` reports `SSH_PRIVATE_KEY  len=398 lines=7 empty=false pem_type="OPENSSH PRIVATE KEY"`, and lucas42 reports that it looks the same as the production value. lucas42/lucos_backups#442 records how it got there: `rotate-ssh-key.sh` writes the production key into the development store as well.
3. **`restart: always` brings it back unattended.** Nobody started a dev backups instance on purpose. A Docker runtime restart revived every container that had ever been brought up on the laptop.

Its retention rule matched production's only because `toKeep` hasn't changed since 2024-11. A dev build carrying an in-progress retention change, or running on a machine with a wrong clock, would have deleted whatever its own rule said.

### The collision, and why it looked like something else

The two prune runs worked through the same hosts within seconds of each other. Whenever one run listed or removed a tree while the other was deleting it, the first saw entries vanish. GNU `find` exits 1 on an entry that disappears mid-walk. BusyBox `rm -rf` on aurora reports `unable to remove … No such file or directory`, and at the end fails on its own top-level directory. Either way the production run failed and reported the misleading `host unreachable`.

The earlier occurrences on lucas42/lucos_backups#365 had been put down to the job racing itself. That was never possible: `prune-backups.py` handles one host at a time, and lists each host before deleting anything there. The reading came from log ordering, and stdout is buffered while stderr is not, so the interleaving in `docker logs` doesn't reflect execution order. Two observations finally separated the hypotheses: a deletion timestamped **after** the production process had exited, and an `rm -rf` whose own directory vanished under it.

### Why monitoring couldn't see the actor

The dev copy reports to a local Loganne and schedule-tracker (`localhost:8019`, `172.17.0.1:8024`). Nothing in production records its runs. The only signals were second-order: the production run's failures, plus `lucos_backups`-UA requests to configy from an address where no production backups instance runs.

### Exposure beyond pruning

The dev copy's `refresh-tracking` (hourly, read-only `docker` listings over SSH) would have run against every host too. Its `create-backups` could have `docker pause`d production containers and written snapshots alongside the real run, including the same-day `rm -rf "{final}" && mv "{partial}" "{final}"` of a snapshot directory. No sign of that was found: there was no off-schedule `db-reachable` alert on 11-21 or 11-22. But without the laptop's logs it can't be ruled out (TBD).

---

## What Was Tried That Didn't Work

- **"The dev environment can't authenticate: its key is a placeholder."** The SRE parsed the dev `.env` line by line, saw only the first line of the 7-line key (`-----BEGIN OPENSSH PRIVATE KEY-----`, 35 characters), and published "a 36-character placeholder" on lucas42/lucos_backups#365. That ruled out the actual culprit for about an hour. The tool built for exactly this, `creds-shape`, would have reported the true shape. The SRE persona now requires credential-content claims to come from `creds-shape` and to be quoted.
- **A duplicate cron entry in the production container.** `cat /etc/crontabs/* ; crontab -l` printed the `prune-backups` line twice, but both read the same file. One crontab, one `crond`, one `Pruning Backups...` per day.
- **`docker events` for duplicate pauses.** It returned nothing, even for a run known to have paused ten containers, so it holds no usable history here and is no evidence either way.
- **sshd auth logs on xwing.** Neither the SRE's nor the sysadmin's account can read them (not in `adm` or `systemd-journal`), and `journalctl` printing "No entries" is not a negative. These logs would name the source address and key fingerprint directly. TBD: lucas42 to run the query.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Stop storing the production backups key in the development store, and rotate it | lucas42/lucos_backups#442 | Open (High), with lucos-security |
| *Considered and rejected:* make a non-production lucos_backups refuse to act on production hosts | lucas42/lucos_backups#476 | Closed by lucas42: a development version whose core logic depends on an `environment == production` flag defeats the point of having one. Credential separation (lucas42/lucos_backups#442) is the control |
| Make `prune-backups` tolerate vanishing entries and report errors honestly (no false `host unreachable`) | lucas42/lucos_backups#365 | Open |
| Verify aurora's retention ladder | TBD (lucas42, via the backups UI) | Pending |
| Router nightly report rejected since 2026-05-15 (found during this incident) | lucas42/lucos_router#114 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

A production-capable SSH private key was present in a development credential store and on a laptop. This report gives only its shape, never its value or fingerprint. Backup filenames from the evidence (photo file names and one-off archive names) have been left out, and only counts are given.
