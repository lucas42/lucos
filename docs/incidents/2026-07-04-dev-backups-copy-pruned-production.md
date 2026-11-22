# Incident: A development copy of lucos_backups pruned the production backup hosts

| Field | Value |
|---|---|
| **Date** | 2026-07-04 to 2026-11-22 (intermittent) |
| **Duration** | 14 days of activity between 2026-07-04 and 2026-11-22 (07-04, 07-05, 07-06; 08-18, 08-20, 08-21, 08-23, 08-25, 08-27, 08-28, 08-30; 10-20; 11-21, 11-22), per the dev container's own log. `prune-backups` red from 04:51 to 18:11 UTC on 11-22 (13h20m), cleared by an ad-hoc rerun |
| **Severity** | Data risk / Security |
| **Services affected** | lucos_backups (the backup copies on xwing, aurora, salvare and avalon) |
| **Detected by** | SRE ops check (the `prune-backups` monitoring check) |

---

## Summary

A development copy of `lucos_backups` on lucas42's laptop ran its scheduled jobs against the **production** backup hosts whenever the laptop happened to be running it: 14 days between 2026-07-04 and 2026-11-22, the last session brought back by an OrbStack restart, since the compose file sets `restart: always`. It could do so because the development store holds a working SSH key, which lucas42 reports looks the same as the production key (lucas42/lucos_backups#442), and because every copy, by design, takes its host list from configy. On each of those days it archived 24–27 production volumes at 03:25 UTC and pruned at 04:49 UTC, alongside the real prune. When the two prunes collided, the production run failed, which is every occurrence on lucas42/lucos_backups#365 since July. Every dev prune count matches what the retention rule makes due that day. Every retained archive carrying a dev-run date is production's own later overwrite. The retention ladder is intact on xwing and salvare. Aurora's backups were not verified (see "Not verified"). The dev copy reported only to local services, so production monitoring never saw its runs, only the damage they did to the real ones.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-15 → | Not part of this incident, but found during it: lucos_router's nightly schedule-tracker report has been rejected (HTTP 400) since the v2 migration (lucas42/lucos_router#114) |
| 2026-07-04 03:25 | **First dev-copy session in its log.** `create-backups` archives 21 avalon + 3 xwing production volumes (03:25–03:35), then at 04:49 `prune-backups` deletes 747. 07-04 was a two-cohort day (06-27 and 05-30), and production's two-cohort days just before pruned 745 (06-22) and 748 (06-28), so that is exactly the rule's count. The production run fails on vanished files: lucas42/lucos_backups#365's first occurrence |
| 2026-07-05, 07-06 | Dev sessions (prune 462, 374). Production fails on 07-05 and prunes only 257 on 07-06 against a ~380 baseline |
| 2026-08-18 → 08-30 | Dev sessions on 08-18, 20, 21, 23, 25, 27, 28 and 30 (23 + 3 volumes archived each morning; prunes 396 ×5, 0, 791, plus one failed on aurora's photos snapshot on 08-27). Production fails on 08-20, 08-27 and 08-28 and prunes below baseline on 08-21 (271) and 08-23 (263). The 08-28 occurrence holds the check red 9h36m and is diagnosed as the job racing itself |
| 2026-10-20 | Dev session: 24 + 3 volumes archived (03:25–04:09); prune fails on aurora's photos snapshot |
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
| 2026-11-22 (evening) | lucas42 supplies the dev container's full timestamped log: 14 sessions since 07-04, covering every lucas42/lucos_backups#365 failure date. The actor is confirmed |
| 2026-11-23 04:49 | First scheduled `prune-backups` run with the dev copy stopped: the check that the incident is over. TBD pending result |

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

### What else the dev copy did to production

**Prunes: within the rule every time.** `Backup.prune()` counts every `rm` it issues for an instance it listed, and `rm -f`/`rm -rf` succeed silently on something already gone, so both copies count a shared cohort. The test is therefore that no single count exceeds what `toKeep` makes due that day. None does. Single-cohort days come to 374 and 396 (production 371–396), two-cohort days to 747 and 791 (production 745–792), the one-year-boundary day 07-05 to 462 (production 525–564 on later such days), and 08-25, with nothing due, to 0. That agrees with a full-history ladder check on xwing and salvare, which found no missing kept instances.

**Backups: unquiesced, then overwritten.** Paused capture entered the code on 2026-09-13 (v1.4.53). The dev copy deleted snapshot directories (a strategy added 2026-06-10) and was running by 07-04, so its image dates from between those two dates and, assuming the container was never recreated, it never paused anything. That assumption is not verified: the container's creation date and image were not available. Its archives were therefore unquiesced copies of live databases, including the credential and auth stores, written under production's own filenames. The monitoring record fits: production's own pause of photos' Postgres trips `db-reachable` daily at 15:3x, but nothing alerted during the dev runs' 03:25–03:55 windows on 11-21 and 11-22. They did no lasting harm only because production's 15:25 run overwrote them the same day. Every retained archive carrying a dev-run date (`2026-07-06` on xwing and salvare, 24 each; `2026-11-21` and `2026-11-22` on avalon, xwing and salvare, 27 each) has an afternoon mtime from production's run, none from the 03:00–05:00 dev window. Aurora was not checked. Production's 03:25 run no-ops after a recent success, so the two backup runs never overlapped. The dev runs play no part in the daily 15:3x photos alert (lucas42/lucos_backups#466).

**The rest.** Its hourly `refresh-tracking` made read-only `docker` listings over SSH on every host.

**What isn't this incident.** avalon is missing every monthly 6th from 2025-12 to 2026-09 for 8 volumes. A deleter applying our rule never removes 6ths, and the shape matches the 2026-09 disk loss and restore.

### Not verified

The investigation closed without access to the laptop or aurora, so the following remain open. Each is bounded but not settled:

- **Whether the dev `create-backups` paused production containers.** Inferred "no" from the code history and the monitoring record (above). The container's creation date and image, which would confirm it, were not available.
- **Whether it wrote snapshot directories on aurora** (the photos snapshot strategy rsyncs to aurora) and whether those raced production's same-day replacement of that day's snapshot. The laptop log was filtered to archive-creation and prune lines, so it shows neither way.
- **Aurora's retention ladder and archive timestamps.** No agent has aurora access. The residual risk is bounded: the dev copy applied the same `toKeep` rule (unchanged since 2024-11), and its prune counts match that rule on every day, across all hosts combined.
- **Whether there were sessions before 2026-07-04.** The log's first line and the container's creation date weren't captured. 07-04 is the first session in the supplied log, and the first production failure on lucas42/lucos_backups#365.
- **sshd auth-log evidence** (source address and key fingerprint per login). Not obtained. The laptop log identifies the actor regardless, but the claim that its key is the production key rests on lucas42's report that the two look the same.
- **The few seconds between the dev log's prune end (04:50:26, laptop clock) and xwing's directory changes (04:50:30–33) on 11-22.** That fits the dev copy only if the laptop's VM clock ran slightly slow, which wasn't checked.

---

## What Was Tried That Didn't Work

- **"The dev environment can't authenticate: its key is a placeholder."** The SRE parsed the dev `.env` line by line, saw only the first line of the 7-line key (`-----BEGIN OPENSSH PRIVATE KEY-----`, 35 characters), and published "a 36-character placeholder" on lucas42/lucos_backups#365. That ruled out the actual culprit for about an hour. The tool built for exactly this, `creds-shape`, would have reported the true shape. The SRE persona now requires credential-content claims to come from `creds-shape` and to be quoted.
- **A duplicate cron entry in the production container.** `cat /etc/crontabs/* ; crontab -l` printed the `prune-backups` line twice, but both read the same file. One crontab, one `crond`, one `Pruning Backups...` per day.
- **`docker events` for duplicate pauses.** It returned nothing, even for a run known to have paused ten containers, so it holds no usable history here and is no evidence either way.
- **The laptop log's `pause` filter.** `grep -E 'pause'` found nothing, but it's case-sensitive and production logs `Paused for …`, so it couldn't have matched. Whether the dev copy paused anything was settled instead from the code history and the monitoring record (above).
- **sshd auth logs on xwing.** Neither the SRE's nor the sysadmin's account can read them (not in `adm` or `systemd-journal`), and `journalctl` printing "No entries" is not a negative. These logs would name the source address and key fingerprint directly. Superseded by the laptop's own log, which identifies the actor directly.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Stop storing the production backups key in the development store, and rotate it | lucas42/lucos_backups#442 | Open (High), with lucos-security |
| *Considered and rejected:* make a non-production lucos_backups refuse to act on production hosts | lucas42/lucos_backups#476 | Closed by lucas42: a development version whose core logic depends on an `environment == production` flag defeats the point of having one. Credential separation (lucas42/lucos_backups#442) is the control |
| Make `prune-backups` tolerate vanishing entries and report errors honestly (no false `host unreachable`) | lucas42/lucos_backups#365 | Open |
| Router nightly report rejected since 2026-05-15 (found during this incident) | lucas42/lucos_router#114 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

A production-capable SSH private key was present in a development credential store and on a laptop. This report gives only its shape, never its value or fingerprint. Backup filenames from the evidence (photo file names and one-off archive names) have been left out, and only counts are given.
