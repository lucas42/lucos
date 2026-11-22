# Incident: A development copy of lucos_backups pruned the production backup hosts

| Field | Value |
|---|---|
| **Date** | 2026-07-04 to 2026-11-22 (intermittent) |
| **Duration** | Dev-copy activity on 14 days since 2026-07-04; `prune-backups` check red for 13h20m on 2026-11-22 |
| **Impact** | No user-facing outage. A monitoring check whose message misled diagnosis for four and a half months, and an unmanaged process with root-equivalent access deleting and copying production backups |
| **Severity** | Data risk / Security |
| **Services affected** | lucos_backups (the backup copies on xwing, aurora, salvare and avalon) |
| **Detected by** | SRE ops check (the `prune-backups` monitoring check) |

---

## Summary

**What happened.** A development copy of `lucos_backups` on lucas42's laptop ran its scheduled jobs against the **production** backup hosts on 14 days between 2026-07-04 and 2026-11-22. That was whenever the laptop happened to be running it; the last session was most likely brought back by an OrbStack restart, which revives containers because the compose file sets `restart: always`. Each morning it archived 24–27 production volumes at 03:25 UTC and pruned old backups at 04:49 UTC, at the same moment as the real prune job. When the two prunes collided, the production run failed with a misleading `host unreachable`. That explains every failure recorded on lucas42/lucos_backups#365 since July.

**Why it could happen.** The development credential store holds a working SSH key, which lucas42 reports looks the same as the production key (lucas42/lucos_backups#442). Every copy of the system, by design, takes its list of hosts from configy. Nothing else stood between a laptop container and the production machines.

**What it did and didn't damage.** No evidence of lost backups on any host that could be checked. Every dev prune count matches what the retention rule allows that day, and every retained archive carrying a dev-run date was overwritten by production's own run. Aurora could not be checked (see "Not verified"). The dev copy reported only to services on the laptop, so production monitoring never saw its runs, only the damage they did to the real ones.

**Terms used below.** The *retention rule* (`toKeep` in `src/classes/backup.py`) keeps every daily backup for a week, backups whose day of the month is divisible by 6 for five weeks, the 6th of each month for a year, and 6 January thereafter. The *retention ladder* is the set of backups that rule keeps. A *cohort* is the set of backups from one date, across all volumes and repositories, which becomes prunable together. A *two-cohort day* is one when two dates fall off the ladder at once. An *unquiesced* backup is one taken while the database was still being written to, so it may not be internally consistent.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-07-04 03:25 | **First dev-copy session in its log.** `create-backups` archives 21 avalon + 3 xwing production volumes (03:25–03:35), then at 04:49 `prune-backups` deletes 747. 07-04 was a two-cohort day (06-27 and 05-30), and production's two-cohort days just before pruned 745 (06-22) and 748 (06-28), so that is exactly the rule's count. The production run fails on vanished files: lucas42/lucos_backups#365's first occurrence |
| 2026-07-05, 07-06 | Dev sessions (prune 462, 374). Production fails on 07-05 and prunes only 257 on 07-06 against a ~380 baseline |
| 2026-08-18 → 08-30 | Dev sessions on 08-18, 20, 21, 23, 25, 27, 28 and 30 (23 + 3 volumes archived each morning; prunes 396 ×5, 0, 791, and one failed on aurora's photos snapshot on 08-27). Production fails on 08-20, 08-27 and 08-28 and prunes below baseline on 08-21 (271) and 08-23 (263). The 08-28 occurrence holds the check red 9h36m and is diagnosed as the job racing itself |
| 2026-10-20 | Dev session: 24 + 3 volumes archived (03:25–04:09); its prune fails on aurora's photos snapshot |
| 2026-11-20 16:48:57 | First requests from dev-looking clients reach production from the home network address: a Java client (16:48:57), `lucos_creds_configy_sync` (16:48:59), `lucos_dns_sync` (17:00:01). This is consistent with an OrbStack restart at about this time |
| 2026-11-20 17:03:00 | First `lucos_backups` user-agent request from the home network address (configy `/hosts` and `/volumes`), at exactly the job's hourly `refresh-config` slot. There were none in the router log from 11-15 to 11-19 |
| 2026-11-21 04:49 | Both `prune-backups` runs collide. The production run's `rm -rf` of aurora's expired photos snapshot loses its own top-level directory mid-run (1,484 ENOENT lines), and xwing's file listing sees files vanish. The run reports `Prune failed for: aurora.lan/lucos_photos_photos, xwing.s.l42.eu (host unreachable)` |
| 2026-11-22 04:49–04:50 | Same again (448 ENOENT lines on aurora). xwing is skipped by the production run, yet xwing's directories are modified at 04:50:30 and 04:50:33, after the production run had finished at 04:50:26.94 |
| 2026-11-22 04:51 | `prune-backups` turns red (two consecutive failed runs) |
| 2026-11-22 ~16:30 | SRE ops check finds a second deleter. One probe was wrong and wrongly ruled out the development environment (see "What Was Tried That Didn't Work") |
| 2026-11-22 17:03:00 | The dev copy's last `lucos_backups` request from the home network. None arrives at the 18:03 slot, so it was stopped in between |
| 2026-11-22 ~17:30 | lucas42 suggests the OrbStack dev copies. SRE confirms that the dev store holds a real key and that the router-log onset matches |
| 2026-11-22 ~17:45 | Retention ladder verified intact on xwing and salvare. lucas42 stops all dev containers on the laptop (before 18:03, per the router log) |
| 2026-11-22 ~18:05 | Ad-hoc `prune-backups` rerun: clean, 1 instance pruned (on aurora) |
| 2026-11-22 18:11:39 | `prune-backups` healthy; dashboard 56/56 |
| 2026-11-22 after 18:11 | lucas42 supplies the dev container's full timestamped log (exact time not recorded): 14 sessions since 07-04, covering every lucas42/lucos_backups#365 failure date. The actor is confirmed |
| 2026-11-23 04:49 | First scheduled `prune-backups` run with the dev copy stopped, as a check that the incident is over. **Pending: to be recorded before this report is finalised** |

---

## Analysis

### A development copy is a production actor

Three things combined to make a development container act on production. The second is the one that matters.

1. **A development copy runs the same logic as production, by design.** `src/utils/config.py` fetches hosts and volumes from `https://configy.l42.eu`, and the crontab is the same in every environment: `create-backups` at 03:25 and 15:25, `prune-backups` at 04:49, and `refresh-tracking` hourly. That's deliberate. A development version whose core logic branches on `ENVIRONMENT` wouldn't be worth having (see Follow-up Actions). So what keeps a dev copy off the production hosts is not its configuration but its lack of a credential the hosts accept.
2. **The development store holds a working key, and that is the defect.** `creds-shape lucos_backups development` reports `SSH_PRIVATE_KEY  len=398 lines=7 empty=false pem_type="OPENSSH PRIVATE KEY"`, and lucas42 reports that it looks the same as the production value. lucas42/lucos_backups#442 records how it got there: `rotate-ssh-key.sh` writes the production key into the development store as well. What that key unlocks is more than backups. The `lucos-backups` user is in the `docker` group on avalon, xwing and salvare, which makes it root-equivalent on those hosts. (lucos-security verified that on 2026-10-19; aurora wasn't checked.)
   This is an *integrity* failure, not only a *confidentiality* one. Earlier reasoning about production values in the development store (lucas42/lucos_creds#603, and the original severity on lucas42/lucos_backups#442) asked who could *read* them. Because lucas42/lucos_agent_coding_sandbox#102 already gives agents equivalent access, the extra exposure looked small. What happened here is that a development copy *used* the value and so became a production actor. #102 does nothing to reduce that. The question for any dev value identical to production's is therefore also whether a development copy of that system would act with it.
3. **`restart: always` brings it back unattended.** Nobody started a dev backups instance on purpose. A Docker runtime restart revived every container that had ever been brought up on the laptop. `restart: always` is the estate convention, so dropping it from dev isn't a free fix either. That's another reason credential separation is the right control.

Its retention rule matched production's only because `toKeep` hasn't changed since 2024-11. A dev build carrying an in-progress retention change, or running on a machine with a wrong clock, would have deleted whatever its own rule said.

### The collision, and why it looked like something else

The two prune runs worked through the same hosts within seconds of each other. Whenever one run listed or removed a tree while the other was deleting it, the first saw entries vanish. GNU `find` exits 1 on an entry that disappears mid-walk. BusyBox `rm -rf` on aurora reports `unable to remove … No such file or directory`, and at the end fails on its own top-level directory. Either way the production run failed. It then reported `host unreachable`, a single fixed label applied to every host that failed for any reason, which pointed every reader at the network.

The earlier occurrences on lucas42/lucos_backups#365 had been put down to the job racing itself. That was never possible: `prune-backups.py` handles one host at a time, and lists each host before deleting anything there. The reading came from log ordering, and stdout is buffered while stderr is not, so the interleaving in `docker logs` doesn't reflect execution order. Two observations finally separated the hypotheses: a deletion timestamped **after** the production process had exited, and an `rm -rf` whose own directory vanished under it.

### Why monitoring couldn't see the actor

The dev copy reports to a local Loganne and schedule-tracker (`localhost:8019`, `172.17.0.1:8024`). Nothing in production records its runs. The only signals were second-order: the production run's failures, plus `lucos_backups`-UA requests to configy from an address where no production backups instance runs. The vanished-entry errors were the decisive one. Production's prune can't race itself, so they could only mean a second deleter, and the rescoped #365 fix makes them a distinct, honestly-labelled failure rather than suppressing them.

The other development clients that appeared on 11-20 look benign. Going by their code on `origin/main`, dev `lucos_dns_sync` reads production configy and writes zone files locally. Dev `lucos_creds_configy_sync` reads production configy and writes over SSH to `lucos_creds`, a compose service name that Docker's network DNS resolves to the laptop's own creds container. That wasn't checked live on the laptop. A dev Java client's 5,840 requests to media-api over three days were all rejected with 401. On that reading, lucos_backups was the only observed development client acting on production.

### What else the dev copy did to production

**Prunes: within the rule every time.** `Backup.prune()` counts every `rm` it issues for an instance it listed, and `rm -f`/`rm -rf` succeed silently on something already gone, so both copies count a shared cohort. The test is therefore that no single count exceeds what the retention rule makes due that day. None does:

| Kind of day | Dev count | Production on comparable days |
|---|---|---|
| One cohort due | 374; 396 ×5 | 371–396 |
| Two cohorts due (07-04, 08-28) | 747, 791 | 745–792 |
| One cohort plus the 2025-07-06 backups turning a year old (07-05) | 462 | 525–564 on later one-year days |
| Nothing due (08-25) | 0 | — |

That agrees with a full-history ladder check on xwing and salvare, which found no missing kept backups.

**Backups: unquiesced, then overwritten.** Paused capture, which briefly pauses a database's container so its backup is consistent, entered the code on 2026-09-13 (v1.4.53). The dev copy deleted snapshot directories (a strategy added 2026-06-10) and was running by 07-04, so its image dates from between those two dates. Assuming the container was never recreated, it never paused anything. Its archives were therefore unquiesced copies of live databases, including the credential and auth stores, written under production's own filenames. The monitoring record fits: production's own pause of photos' Postgres trips `db-reachable` daily at 15:3x, but nothing alerted during the dev runs' 03:25–03:55 windows on 11-21 and 11-22. The dev archives did no lasting harm only because production's 15:25 run overwrote them the same day. Every retained archive carrying a dev-run date (`2026-07-06` on xwing and salvare, 24 each; `2026-11-21` and `2026-11-22` on avalon, xwing and salvare, 27 each) has an afternoon mtime from production's run, and none from the 03:00–05:00 dev window. Production's 03:25 run no-ops after a recent success, so the two backup runs never overlapped. The dev runs play no part in the daily 15:3x photos alert (lucas42/lucos_backups#466).

**The rest.** Its hourly `refresh-tracking` made read-only `docker` listings over SSH on every host. The laptop log was filtered to archive and prune lines, so that's the extent of what is *known*. See "Not verified".

**What isn't this incident.** avalon is missing every monthly 6th from 2025-12 to 2026-09 for 8 volumes. The retention rule never removes 6ths, so a deleter applying it couldn't have done this, and the shape matches the 2026-09 disk loss and restore (inferred from the shape, not traced). Separately, lucos_router's nightly schedule-tracker report has been rejected since 2026-05-15; it was found during this investigation but is unrelated (see Follow-up Actions).

### Not verified

The investigation closed without access to the laptop or aurora, so the following remain open. Each is bounded but not settled:

- **The full set of commands the dev copy ran over SSH.** There's no evidence of anything beyond its scheduled jobs, but the laptop log was filtered and the hosts' sshd logs were never read. Given the key's root-equivalent reach, "no evidence of other commands" is as far as this report can go.
- **Whether the dev `create-backups` paused production containers.** Inferred "no" from the code history and the monitoring record (above). The container's creation date and image, which would confirm it, weren't available.
- **Whether it wrote snapshot directories on aurora** (the photos snapshot strategy rsyncs to aurora) and whether those raced production's same-day replacement of that snapshot. The filtered log shows neither way.
- **Aurora's retention ladder and archive timestamps.** No agent has access. lucos-system-administrator attempted it on 2026-11-22: aurora's sshd answered through xwing, but `lucos-agent` isn't authorised (`Permission denied (publickey,password,keyboard-interactive)`). The residual risk is bounded: the dev copy applied the same retention rule (unchanged since 2024-11), and its prune counts, which cover all hosts combined, match that rule on every day.
- **Whether there were sessions before 2026-07-04.** The log's first line and the container's creation date weren't captured. 07-04 is the first session in the supplied log, and the first production failure on lucas42/lucos_backups#365.
- **sshd auth-log evidence** (source address and key fingerprint per login). Not obtained on any host. The laptop log identifies the actor regardless, but two things rest on lucas42's report that the keys look the same: that the dev store's key is the production key, and that the container presented the stored key rather than another one.
- **The few seconds between the dev log's prune end (04:50:26, laptop clock) and xwing's directory changes (04:50:30–33) on 11-22.** That fits the dev copy only if the laptop's VM clock ran slightly slow, which wasn't checked.

---

## What Was Tried That Didn't Work

- **"The dev environment can't authenticate: its key is a placeholder."** The SRE parsed the dev `.env` line by line, saw only the first line of the 7-line key (`-----BEGIN OPENSSH PRIVATE KEY-----`, 35 characters), and published "a 36-character placeholder" on lucas42/lucos_backups#365. That ruled out the actual culprit for about an hour. The tool built for exactly this, `creds-shape`, would have reported the true shape. The SRE persona now requires credential-content claims to come from `creds-shape` and to be quoted.
- **A duplicate cron entry in the production container.** `cat /etc/crontabs/* ; crontab -l` printed the `prune-backups` line twice, but both read the same file. One crontab, one `crond`, one `Pruning Backups...` per day.
- **`docker events` for duplicate pauses.** It returned nothing, even for a run known to have paused ten containers, so it holds no usable history here and is no evidence either way.
- **The laptop log's `pause` filter.** `grep -E 'pause'` found nothing, but it's case-sensitive and production logs `Paused for …`, so it couldn't have matched. Whether the dev copy paused anything was judged instead from the code history and the monitoring record (above).
- **sshd auth logs.** Neither the SRE's nor the sysadmin's account can read them on xwing or salvare (not in `adm` or `systemd-journal`), and `journalctl` printing "No entries" is not a negative. Aurora's weren't tried. These logs would name the source address and key fingerprint directly. The laptop's own log later identified the actor instead.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Stop storing the production backups key in the development store, and rotate it. Rotate with an overlap: add the new public key to every host's `authorized_keys`, redeploy, then remove the old one. That last step is what revokes the exposed key, and on aurora (QNAP BusyBox, behind xwing) it's a manual step. Deleting the dev copy doesn't revoke a key that has already been read | lucas42/lucos_backups#442 | Open (Ready, High). Owner lucos-system-administrator; writing the production key is lucas42's step |
| Sweep for other production-grade credentials readable from the development store, and weigh each by whether a development copy would *act* with it, not only by who can read it (comment of 2026-11-22). A candidate lucos_creds-side warning when a simple credential's development value equals its production value is proposed there, not filed | lucas42/lucos_creds#603 | Open |
| *Considered and rejected:* make a non-production lucos_backups refuse to act on production hosts | lucas42/lucos_backups#476 | Closed by lucas42: a development version whose core logic depends on an `environment == production` flag defeats the point of having one. Credential separation (lucas42/lucos_backups#442) is the control |
| Make `prune-backups` name its failures honestly. A vanished entry becomes its own failure cause ("entries deleted concurrently by another process"), and the run keeps pruning the rest. It is neither suppressed with `-ignore_readdir_race` nor downgraded to a log line. Every other failure gets a per-host cause, with `host unreachable` reserved for a failed SSH connection. The original "benign race" premise is withdrawn | lucas42/lucos_backups#365 | Open, rescoped 2026-11-22 |
| Router nightly schedule-tracker report rejected since 2026-05-15 (found during this investigation, unrelated) | lucas42/lucos_router#114 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

A production-capable SSH private key, root-equivalent on three hosts through the `docker` group, was present in the development credential store. That made it readable by every agent persona with development-store access since at least 2026-10-19, so it may appear in past agent transcripts. It was also copied to any machine that pulled lucos_backups' development `.env`, including the laptop. That reach is why the key needs rotating whatever the sshd logs would have shown. This report gives only the key's shape, never its value or fingerprint. Backup filenames from the evidence (photo file names and one-off archive names) have been left out, and only counts are given.
