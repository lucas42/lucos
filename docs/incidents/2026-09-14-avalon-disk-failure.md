# Incident: avalon's single disk failed — estate-wide outage and emergency data rescue

> **DRAFT — the incident is not yet resolved.** Sections marked **TBD** are to be completed once OVH has replaced the disk and avalon has been rebuilt and restored. Source issue: **lucas42/lucos#294**.

| Field | Value |
|---|---|
| **Date** | 2026-09-14 |
| **Duration** | Onset ~07:55 UTC on 2026-09-14. **Ongoing:** avalon has been out of service since ~19:21 UTC that day. End time TBD, pending rebuild. |
| **Severity** | Complete outage (every avalon-hosted service) + data risk |
| **Services affected** | Everything hosted on avalon, which is nearly the whole estate. That includes aithne (login), contacts, eolas, arachne, media (metadata, manager, seinn, weightings), photos, locations, notes, creds, worlds, backups, loganne, schedule-tracker, monitoring, the `l42.eu` router and DNS primary. Services on xwing/salvare kept running, but lost their dependencies on avalon. |
| **Detected by** | Monitoring alerts from ~07:55 UTC (delivered by email). First acted on by an SRE ops check at 12:15 UTC. |

---

## Summary

avalon runs every one of its services from a **single spinning hard disk with no RAID**, an HGST HUS726020ALA610, serial K5H8E1BA. On 2026-09-14 that disk began failing. Reads took seconds each instead of ~20 ms, the kernel logged a steadily rising stream of I/O errors, and services across the estate degraded from ~07:55 UTC, until the host became unmanageable at ~19:21. Nobody acted on the alerts for about 4h20m.

Once engaged, the team avoided anything that would write to the disk. lucas42 booted the server into OVH rescue mode that evening, and the data was copied from a read-only mount to xwing, then to salvare. **Every critical database was recovered and verified as a working database**, including lucas42's lucos_worlds edits up to 00:16 UTC on the 14th, which no backup contained. One database file, media_metadata, had unreadable sectors: it was repaired, with only 2 rows restored from the previous day's backup.

The disk now awaits replacement by OVH, and avalon a rebuild and restore. **(TBD: resolution.)**

---

## Timeline

All times UTC.

| Time | Event |
|---|---|
| 2026-09-13 15:25–15:34 | The last off-host backup run completes. The real daily run happens in the afternoon, not overnight (see "Backup exposure"). |
| 2026-09-14 00:16 | lucas42's last lucos_worlds edit. It's in no backup. |
| 03:25 | The `create-backups` slot skips as "fresh", per the code; not read from logs. |
| 06:20 | A lucos_docker_health avalon alert. Possibly an early sign; not confirmed. |
| 07:09–07:45 | The normal daily deploy burst (~25 deploys to avalon). 07:09:51 is the DNS secondary's last successful contact with the avalon primary. |
| **07:55** | **The first `/_info` timeouts** (lucos_media_metadata_api), then waves across the estate through the morning. Alert emails go out. |
| **12:15** | **An SRE ops check finds 10 systems failing and 3 unknown**, and begins diagnosis. This is the first action on the incident. |
| 12:16–12:31 | Load average ~56 on 4 cores. I/O pressure at 99.8%. `/proc/diskstats` shows reads of ~4–10 s each against a lifetime average of 20.1 ms. `df`, `ps` and `dmesg` hang. |
| 12:24 | lucas42/lucos#294 filed. Escalated as P1 and pushed to lucas42. |
| ~12:37 | `/sys/block/sda/device/ioerr_cnt` is climbing (6 errors in 8 s). **Failing disk identified.** Advice changes to "do NOT reboot" (see "Response"). |
| 12:45–13:00 | Emergency live copy of `lucos_aithne_credential_store`: succeeds and verifies. |
| 13:01–15:45 | Emergency `pg_dumpall` of contacts, then eolas: **0 bytes in 58 min and 1h45m.** Stopped. |
| 15:25 | The daily `create-backups` run starts. It progresses slowly, delivering 4 small archives to xwing between 16:54 and 17:58, then stalls. |
| ~15:44 | lucos_monitoring's own API starts failing. The estate has no working monitoring from here on. |
| 15:52 | The disk error counter reaches 9,143 (5,324 at ~12:37). Load average 205. |
| ~19:21 | **avalon wedges.** The kernel still answers ping and TCP, but SSH, the router and aithne stop responding. |
| **22:38** | **lucas42 reboots avalon into OVH rescue mode.** |
| 22:44–23:07 | `/dev/sda2` is mounted `ro,norecovery`, and the critical volumes are copied to xwing. The media_metadata tar comes out damaged. |
| 23:08–23:13 | media.sqlite chunked re-read, then a sector-level re-read: 14 of the file's sectors remain unreadable. |
| 23:19–23:45 | SQLite `.recover`, then comparison and repair, give `media.final.sqlite`. |
| 23:47–23:55 | Host config and the unique 2025-01-06 yearly backup set copied. |
| 2026-09-15 00:00 | SMART read: 29 pending, 109 offline-uncorrectable sectors, 15,558 logged errors. |
| 00:09–00:12 | The emergency-backups directory is copied to salvare and verified by sha256. |
| TBD | OVH replaces the disk. |
| TBD | avalon rebuilt, and data restored from the emergency backups. |
| TBD | Services verified end to end, including a triggered backup run. Incident resolved. |

---

## Analysis

### Root cause: a failing disk on a host with no redundancy (confirmed)

This is **confirmed** by three independent sources:
- **The kernel**, via `ioerr_cnt`: 5,324 → 10,000+ errors on the 14th, still rising in rescue mode.
- **Direct reads that failed**: `tar` reported `media.sqlite: File shrank … padding with zeros`, and 14 specific 512-byte sectors never read.
- **The drive's own SMART data**:
  ```
  197 Current_Pending_Sector   29
  198 Offline_Uncorrectable    109
    5 Reallocated_Sector_Ct    1
  ATA Error Count: 15558 (most recent: UNC, uncorrectable read)
  SMART overall-health self-assessment: PASSED
  ```
  The overall verdict still says "PASSED" only because no attribute has crossed its manufacturer threshold. That verdict would not have warned anyone.

What made this an outage rather than an inconvenience:
- **No redundancy.** avalon has one disk (`/proc/mdstat` shows no arrays), so a failing disk takes down every service and every database at once.
- **Concentration.** avalon hosts nearly every service, including the alerting chain and the DNS primary.

I don't know when the drive's damage began accumulating. SMART's counts carry no dates, and the kernel counter runs only since boot, 191 days earlier.

### Detection and response: alerts were delivered, then not acted on for 4h20m

- Monitoring detected the degradation correctly from ~07:55.
- lucos_mail_smtp's log, read off the rescue mount, shows **223 alert emails relayed `status=sent` to Google's MX** between 07:00 and 12:59 (3 deferred, 0 bounced). Inbox placement isn't visible to us.
- Nothing acted on them until an SRE ops check at 12:15. That's the alert-to-action gap recorded in lucas42/lucos#290. This occurrence meets that ticket's trigger 2 (a real outage with data at risk).
- **Why the delay mattered:** by 12:15 the disk was too degraded for normal copies. Database dumps produced nothing. Acting at ~08:00 might have allowed clean dumps, but that is plausible, not established.
- **Nothing watches disk health.** An I/O error rate check would have been the most direct early warning, if the errors built up gradually. That's unknown, as noted above. Tracked as lucas42/lucos_docker_health#118.
- **When avalon itself fails, the alerting chain fails with it.** Monitoring, schedule-tracker, loganne and mail all run on avalon. On the 14th the host only degraded, so alerts still went out until ~15:44. A harder failure would have sent nothing. Tracked as lucas42/lucos#295.

### Backup exposure: the newest backup was from the previous afternoon, not that night

- The newest off-host backup was **2026-09-13 15:25**. `create-backups` was designed (lucas42/lucos_backups#225) with **03:25 as the primary run and 15:25 as a retry**. In practice the 03:25 run always skips: a 20-hour freshness window, measured from a `last_success` marker kept in `/var/run` (lost on every container restart), pins the real run to the afternoon. Tracked as lucas42/lucos_backups#415.
- **Consequence:** a 03:25 run on the 14th would very probably have captured lucas42's evening worlds edits. (I haven't confirmed the disk was healthy at 03:25, but onset was hours later.) Instead they survived only because the failing disk could still be read in rescue mode.
- Today's 15:25 run did progress. I had predicted it would fail within seconds, and was wrong. But it was reading and writing a failing disk, and appears to be what caused the 18:14 surge in reads, after which services got worse.

### Data recovery

- **Method:** OVH rescue mode (network boot). `/dev/sda2` was mounted `ro,norecovery`, meaning read-only with no ext4 journal replay, so nothing was ever written to the failing disk. Each volume was streamed as `tar --numeric-owner` to xwing.
- **Verification:** each copy was checked for completeness (`gzip -t` plus a full listing), then **restored and checked with its own database engine**. That meant MariaDB 11.4.12 for worlds, Postgres 16 and 17 for contacts and eolas, pgvector for photos, and `PRAGMA integrity_check` for the SQLite stores.
- **The worlds database** recovered cleanly. Its activity log runs to 00:16:29, with 23 actions after the last backup.
- **media_metadata:**
  - The plain copy was ~96% zeros: `tar` gives up after the first unreadable sector.
  - A chunked re-read (4 KiB blocks, then 512-byte sectors) left **14 unreadable sectors**.
  - SQLite `.recover` rebuilt the database. Compared row by row with the 13 Sep backup, it had lost exactly **2 `track` rows**, restored from that backup, and no tag data, and it held 99 newer tag rows.
  - The resulting `media.final.sqlite` passes `integrity_check`.
- **Where it all is:** in `~lucos-agent/emergency-backups-2026-09-14/` on **xwing and salvare**. That directory's `README.md` is the restore guide: which file to restore each volume from, how each copy was taken and verified, the media repair step by step, what wasn't copied and why, and sha256 checksums.
- **What was lost:** the 14 unreadable sectors, whose only known effect is the 2 `track` rows, restored from 13 Sep. Everything not copied was either regenerable, duplicated elsewhere, or deliberately skipped (photo originals, re-syncable from the Android app per lucas42). **No full-disk image was taken**, by lucas42's decision.

### DNS: a deadline created by the outage

avalon is the primary nameserver for `l42.eu`, `s.l42.eu`, `lukeblaney.co.uk`, `rowanblaney.co.uk` and `tfluke.uk`. The secondary last synced at 2026-09-14 07:09:51, and the SOA `expire` is 28 days. **Unless a primary is back by 2026-10-12 07:09:51 UTC, all five zones stop resolving.** lucas42 expects the rebuild to land well before then, so no follow-up issue has been filed. The deadline is recorded on lucas42/lucos#294.

### Response: corrections made along the way

These are recorded so the report shows what actually happened, not a tidied version:
- **Reboot advice changed three times.** At 12:2x the advice was that a reboot was "the likely lever", made before the failing disk was identified. At ~12:38 it became "do NOT reboot", once the disk was found. At 19:25, with the host unmanageable, it became "reboot into *rescue mode*".
- **The emergency copy list was assembled from memory** and missed `lucos_locations_store`, which configy rates *huge*. The coordinator caught this by cross-checking configy, and it was copied. The SRE instructions now say to build the list from configy (lucas42/lucos_claude_config commit `0406391`).
- **The copy script published 0-byte files under their final names** when the sending side died, because the receiving end can't tell a dead sender from a clean end of stream. Its own log reported the failures correctly. The misleading files were deleted.
- **The damaged media tar passed the archive check.** `tar` pads unreadable ranges with zeros and produces a well-formed archive. It was caught only because the script also surfaced `tar`'s exit code and error output. Restoring each copy into its engine is now an explicit instruction (same commit).
- **The SRE's stop decision was mistakenly reported as "delayed by 1h45m".** In fact lucas42 decided at 15:44:49, a minute before the SRE stopped the copies itself. An instruction change based on that false premise was trimmed back to its valid part (commits `b76ba11` → `f431318`).

---

## What Was Tried That Didn't Work

- **Emergency `pg_dumpall` of contacts and eolas during the degradation:** 0 bytes after 58 min and 1h45m. By then the disk was too slow for the database to produce even its first output.
- **`docker run alpine` on avalon for the first emergency copy:** the image wasn't cached, and the pull from Docker Hub timed out. Switched to an image already on the host, with `--pull=never`.
- **A plain `tar` of `lucos_media_metadata_api_db` in rescue mode:** it stopped at the first unreadable sector and zero-filled the remaining ~96% of `media.sqlite`. Chunked, sector-level re-reads recovered all but 14 sectors.
- **Diagnostic tools on the stalled host:** `ps`, `df`, `lsblk` and `dmesg` hung on the disk. The kernel counters (`/proc/diskstats`, `/proc/pressure/*`, per-cgroup `memory.*`, sysfs `ioerr_cnt`) were what worked.
- **Restarting containers:** deliberately not tried. It would have added I/O to a stalled device and couldn't fix a hardware fault.

---

## Resolution

**TBD.** This section is to be written once OVH has replaced the disk and avalon has been rebuilt and restored. The rebuild and restore procedure lives in lucos-system-administrator's runbook, lucas42/lucos#296. The data comes from the emergency-backups directory described above. Verification must include a **triggered `create-backups` run**, not just green `/_info`s.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| OVH/Kimsufi disk replacement | external support ticket (lucas42) | Waiting on OVH |
| Rebuild avalon and restore from the emergency backups | lucas42/lucos#296 (runbook) | Open (awaiting lucas42's decision) |
| Decide the rebuilt avalon's disk layout (RAID or a second disk?) | lucas42/lucos#296 (runbook, item 2) | Decided: **stays single-disk for now**, as the server is mid-way through a year-long contract. Revisit at renewal. This raises the value of lucas42/lucos_docker_health#118. |
| Make `create-backups` run overnight as designed | lucas42/lucos_backups#415 | Open |
| Decide whether alerting should survive avalon going down hard | lucas42/lucos#295 | Open (decision) |
| Add a host disk-health signal (I/O error rate, optionally SMART) | lucas42/lucos_docker_health#118 | Open (decision) |
| Record this occurrence against the alert-to-action gap | lucas42/lucos#290 (comment) | Done |
| DNS zone expiry deadline (2026-10-12 07:09:51 UTC) | recorded on lucas42/lucos#294; no issue, by lucas42's decision | Monitoring |
| Build emergency copy lists from configy; verify copies as data | lucas42/lucos_claude_config `0406391` | Done |
| Run long production operations under Monitor, not in the foreground | lucas42/lucos_claude_config `f431318` | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] Yes — see note below.

The rescued data includes the **credential store** (`lucos_creds_store`, including its key files), the **aithne credential store**, and host account `authorized_keys` files. They're stored only in `~lucos-agent/emergency-backups-2026-09-14/` on xwing and salvare, where both the home directory and the folder are mode 700. avalon's **SSH host keys** (private) were also copied, on 2026-09-15 at lucas42's request, into `rescue/avalon-ssh-host-keys/` in the same folder, with files at 600. This keeps open the option of reusing them if the rebuild keeps the hostname `avalon` (pending lucas42/lucos#296), and they are to be deleted if it gets fresh keys. `/etc/shadow` and all other private keys were deliberately **not** copied. No credential values appear in this report or in the linked issues.
