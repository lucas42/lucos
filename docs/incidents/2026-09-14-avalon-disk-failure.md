# Incident: avalon's single disk failed — estate-wide outage and emergency data rescue

> **Resolved 2026-09-16. Recovery completed the same evening.** avalon was rebuilt and the estate restored and verified, and all 55 monitored systems are healthy. **The photo originals are recovered** — and the premise on which this report first described that recovery was wrong; see "Photos" below, which is the most important correction in this document. Source issue: **lucas42/lucos#294**; rebuild runbook: **lucas42/lucos#296**.

| Field | Value |
|---|---|
| **Date** | 2026-09-14 |
| **Duration** | Onset ~07:55 UTC on 2026-09-14; avalon unmanageable from ~19:21 that day. Disk replaced and host reinstalled 2026-09-15; services restored through the early hours of 2026-09-16, with verification complete at **03:39 UTC on 2026-09-16**. **About 1 day 20 hours.** The last service, `lucos_mail_smtp`, was restored at 07:31 that morning. **All data recovery finished at 22:34 on 2026-09-16 — about 2 days 15 hours from onset** — the extra day spent recovering photo originals that turned out never to have been lost. |
| **Severity** | Complete outage (every avalon-hosted service) + data risk |
| **Services affected** | Everything hosted on avalon, which is nearly the whole estate. That includes aithne (login), contacts, eolas, arachne, media (metadata, manager, seinn, weightings), photos, locations, notes, creds, worlds, backups, loganne, schedule-tracker, monitoring, the `l42.eu` router and DNS primary. Services on xwing/salvare kept running, but lost their dependencies on avalon. |
| **Still open** | **Nothing.** All data is recovered, including the photo originals, and all 55 monitored systems are healthy. The photos library finished at **2,253 items, fully self-consistent** — every row with its file, every file with its row. What remains is improvement work, not recovery: see "Follow-up Actions". |
| **Data lost** | **None.** Every dataset was recovered, from a backup, from the rescue, or — for three photos — from the volume copy set aside during the restore. The only casualty is 14 unreadable sectors in `media.sqlite`, repaired to 2 rows restored from the previous day. |
| **Detected by** | Monitoring alerts from ~07:55 UTC (delivered by email). First acted on by an SRE ops check at 12:15 UTC. |

---

## Summary

avalon runs every one of its services from a **single spinning hard disk with no RAID**, an HGST HUS726020ALA610, serial K5H8E1BA. On 2026-09-14 that disk began failing. Reads took seconds each instead of ~20 ms, the kernel logged a steadily rising stream of I/O errors, and services across the estate degraded from ~07:55 UTC, until the host became unmanageable at ~19:21. Nobody acted on the alerts for about 4h20m.

Once engaged, the team avoided anything that would write to the disk. lucas42 booted the server into OVH rescue mode that evening, and the data was copied from a read-only mount to xwing, then to salvare. **Every critical database was recovered and verified as a working database**, including lucas42's lucos_worlds edits up to 00:16 UTC on the 14th, which no backup contained. One database file, media_metadata, had unreadable sectors: it was repaired, with only 2 rows restored from the previous day's backup.

Kimsufi replaced the disk on 2026-09-15, and lucas42 reinstalled the host the same evening, on Debian trixie and on the same IPv4 address. The rebuild then surfaced a second finding, independent of the disk: **while avalon is down, no project in the estate can build or deploy at all.** CI fetches every project's credentials from `creds.l42.eu`, which runs on avalon, and a second, unrelated orb defect hard-fails every build against the container mirror, which is also on avalon. That dictates the order of the restore, and both are covered under "Rebuild" below. The estate was restored through the early hours of 2026-09-16 and verified end to end by 03:39 that morning, including a triggered backup run that copied 124 archives to all three destinations. The outbound mail relay followed at 07:31.

**A day of recovery work then went into a dataset that had never been lost.** This report, as first merged, stated that the photo originals were excluded from backups and could only come back from lucas42's phone. That was wrong: aurora had held a complete snapshot throughout, and nobody checked. The consequences — a ten-hour resync, a volume/database mismatch, and a second restore to fix it — are set out under "Photos", which is the section to read if you read only one.

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
| 2026-09-15, by 22:03 | **Kimsufi replace the disk**, and lucas42 reinstalls the host: Debian trixie 13.7, same IPv4 address. He provisions it from his own host-setup notes rather than lucas42/lucos#296's Step 1, so that step is not a record of what was done. |
| 2026-09-15 | avalon serves **freshly generated SSH host keys**, not the rescued ones. Deliberate, and the fallback lucas42/lucos#296's Step 1 allows. Confirmed three ways (live `ssh -vvv`, an isolated `ssh-keyscan`, and the earlier fingerprint report). |
| 23:02 | lucas42 reports host provisioning done, and asks for one service to be deployed as a pipeline test. |
| 23:21 | **`lucos_root` deploys to the rebuilt avalon and answers on `/_info`.** The deploy pipeline works — though not unaided: per lucos-system-administrator's own account on lucas42/lucos#296, the deploy needed a temporary `LUCOS_DEPLOY_ENV_BASE64` project variable holding a placeholder non-secret envfile, removed immediately afterwards, precisely because creds was down. The bypass is the dependency described under "Rebuild". The only failing step is the loganne deploy log, since loganne is also on avalon and not yet up. |
| 23:21–23:28 | The test surfaces a blocker: CI fetches every project's credentials from `creds.l42.eu`, on avalon, for both builds and deploys. lucas42/lucos#296's Step 3 order (`lucos_configy` → `lucos_dns` → `lucos_creds`) therefore cannot run as written. A revised sequence is proposed. |
| 2026-09-16 | lucas42 approves the revised sequence: `lucos_creds` first, via its CI bypass, with its store restored immediately after, then `lucos_configy`, DNS, the router, the firewall and the rest. The sysadmin works down it. (Time not recorded; relayed by team-lead.) |
| 23:33–23:47 | **`lucos_creds` is deployed**, by selective rerun of its last good pre-incident pipeline (CircleCI pipeline 1324, 2026-09-11), and its store restored from the rescue tarball. Three attempts: the first deploys, the second fails at `Deploy using docker compose` mid-restore, the third succeeds at 23:44:06–23:47:25. In the first and third, every real step passed and only "Send deploy log to loganne" failed, loganne being on avalon and not yet up. (Times and step outcomes read from the CircleCI API.) |
| 23:49:17–23:49:34 | **A fresh `lucos_configy` build fails**, at `lucos/build`. This is the second orb defect described under "Rebuild", not the credentials one. |
| 23:52:44–23:55:54 | **`lucos_configy` is deployed** instead by rerunning its 2026-09-08 pipeline (CircleCI pipeline 682). Again, only the loganne step fails. |
| 2026-09-16 | `lucos_backups/init-host.sh` runs successfully, now that creds is up: `/srv/backups` and the `lucos-backups` account exist on avalon (confirmed by team-lead). Next up: `lucos_docker_mirror`, then DNS. |
| 2026-09-16 | **`lucos_dns_bind` fails to start**: `failed to bind host port 0.0.0.0:53/tcp: address already in use`. systemd-resolved (pid 393) already holds port 53 on the fresh trixie install. Found in dockerd's log, which needed lucas42's root access. The container's empty network attachment and its "healthy but unreachable" appearance were consequences of the failed start, not a Docker networking fault. |
| 2026-09-16 | **A second finding in the same logs:** dockerd's resolver times out against `127.0.0.53` for `configy.l42.eu` and `schedule-tracker.l42.eu`, so containers on avalon cannot resolve hostnames. Cause not yet established; lucos-system-administrator is preparing a fix for lucas42 to run, since it needs root. |
| 2026-09-16 | **lucas42 reboots avalon deliberately**, to apply his `resolv.conf` changes. Everything on the host is unreachable for a few minutes. Planned, not a fault — and the first test of whether the rebuilt host brings its services back by itself. |
| 2026-09-16 | **avalon comes back and every already-deployed container restarts on its own**, reporting healthy: the three `lucos_creds` containers, `lucos_configy`, the three `lucos_docker_mirror` containers, `lucos_root_app` and `lucos_dns_sync`. So `docker.service` is enabled at boot on the new install. `lucos_dns_bind` also came back — in the same broken state, running and "healthy" with no network and no published ports, because port 53 was still held at reboot time. |
| 2026-09-16 | **Port 53 freed**, with `DNSStubListener=no`. lucas42's first pass at the change didn't take — the config still read `yes` after the reboot — which is why the reboot alone didn't fix it. `lucos_dns` is redeployed. |
| 00:45 | **The router is back.** `https://l42.eu/_info` answers 200 from outside, serving the restored pre-incident certificates rather than newly issued ones. |
| 01:04 onwards | **Monitoring is deployed and cannot answer.** Every endpoint returns 500 after ~5s, and the container is `Up (unhealthy)`, which also failed its CI deploy job. Cause below. |
| 01:14 | **`lucos_loganne` is back, and monitoring recovers on its own** — API 200, no restart. SMTP is still down, which shows the dominant blocker was loganne's 60s timeouts rather than the refused mail port: a channel that hangs starves reads, one that refuses does not. |
| 01:20–02:09 | **The remaining services are deployed and their volumes restored** — creds, configy, mirror, DNS, router, firewall, then contacts, eolas, photos, worlds, notes, locations, media, arachne, repos and the rest. 33 of avalon's 34 services come back. |
| 02:12–02:15 | The five deploy pipelines that had failed only on their loganne step are re-run and all reach success. `lucos_firewall` is deliberately left, since reapplying firewall rules to win a green tick is the wrong trade with nobody awake to undo it. |
| 02:18:35 | **A `create-backups` run is triggered** as the authoritative verification. It **fails** at ~02:34: `errors=1`, no loganne event, and every `lucos_backups` check still green. The cause is lost with the traceback. |
| 03:24:54 | **A second `create-backups` run is triggered**, output captured to a file. |
| **03:39:58** | **It completes cleanly — "124 archives successfully backed up"**, the same count as every pre-incident run, copied to xwing, salvare and aurora. **Verification complete; the incident is resolved**, apart from `lucos_mail_smtp`. |
| 04:20 | aurora confirmed holding 23 archives dated 2026-09-16, reached through the backups container's own path. |
| 07:31 | **`lucos_mail_smtp` restored** (lucas42/lucos_mail#80). Outbound mail, and therefore alerting, works again. |
| 08:29 | This report merged as lucas42/lucos#297 — describing the photo originals as unrecoverable. Everything below happened afterwards. |
| 10:22–12:46 | **A second, post-fix phone resync runs**, delivering 2,253 originals over 10.2 hours of wall-clock (about 32 minutes of actual transfer; the rest is the phone idling). Two videos are rejected — one zero-byte, one truncated. |
| ~16:00 | **The premise collapses.** Checking `lucos_configy` rather than recalling it shows `lucos_photos_photos` is backed up, and aurora is holding a complete 2026-09-14 snapshot: 2,250 originals, 5,941 files, 11GB. The day's resync was never necessary. |
| 16:17–16:58 | **The photos volume is restored from aurora** by rsync via the xwing gateway. The existing volume is moved aside rather than overwritten, which is what preserves the three unique originals. |
| 17:00 | The restore is verified: the live volume matches the snapshot exactly, 0 differences in either direction. It also exposes the mismatch — the database has been running all day while the files came from a fixed point. |
| ~22:10–22:20 | **The database is restored from the 2026-09-13 nightly** to match the files, on lucas42's diagnosis. Every recorded prediction holds exactly; four consistency counts all reach zero. |
| 22:33 | **The three unique originals are returned** through the normal upload endpoint, from inside the API container. All 201, processed in under 20 seconds, `taken_at` correct to the millisecond from their own EXIF. |
| **22:34** | **The photos library is complete and self-consistent at 2,253 items. All recovery is finished.** |

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
- **What was lost:** the 14 unreadable sectors, whose only known effect is the 2 `track` rows, restored from 13 Sep. Everything not copied was either regenerable, duplicated elsewhere, or deliberately skipped. **No full-disk image was taken**, by lucas42's decision.
- **A correction to the line above, which originally read "…or deliberately skipped (photo originals, re-syncable from the Android app per lucas42)".** The photo originals were skipped during the rescue on the belief that they were not backed up and would be re-synced from the phone. They *were* backed up — aurora held a complete snapshot — so skipping them in the rescue was in fact the right call, but for the wrong reason, and the wrong reason is what then sent the recovery down a ten-hour path. Right answer, wrong model: the kind of near-miss that only shows up when the model is later asked to carry more weight than it can. See "Photos".

### DNS: a deadline created by the outage

avalon is the primary nameserver for `l42.eu`, `s.l42.eu`, `lukeblaney.co.uk`, `rowanblaney.co.uk` and `tfluke.uk`. The secondary last synced at 2026-09-14 07:09:51 and the SOA `expire` is 28 days, so **had the rebuild not landed by 2026-10-12 07:09:51 UTC, all five zones would have stopped resolving** — a silent, estate-wide failure with no issue tracking it, by lucas42's decision that the rebuild would be quicker. It was.

**That deadline is no longer live.** At 00:37:21 on 2026-09-16 the rebuilt primary sent NOTIFYs for all five zones and the secondary logged `zone is up to date` for each, so contact is re-established; the serials match primary to secondary (`l42.eu` 1783445400, `s.l42.eu` 1780876665, `lukeblaney.co.uk` 20, `rowanblaney.co.uk` 17, `tfluke.uk` 25). A successful refresh check resets the expire timer, which clears the deadline — **that last step is my reading of BIND's behaviour rather than something I verified**, because `rndc` is not usable on that container (`rndc.key` exists but the server closes the connection, and `named.conf` has no `controls` stanza).

What the episode exposed is worse than the deadline: the secondary that carried the estate for two days **holds no zone files on disk at all** (lucas42/lucos_dns#135). Had it restarted while the primary was down, there would have been nothing to serve and nothing to transfer from.

### Rebuild: nothing in the estate can build or deploy while `creds.l42.eu` is down

This was found by lucos-system-administrator during the one-service pipeline test on 2026-09-15, and it is the most significant finding of this incident after the disk itself. I have verified the mechanism against `lucas42/lucos_deploy_orb` `main`:

- **Deploys:** `src/commands/deploy.yml` fetches the project's production envfile with `scp … docker-deploy@creds.l42.eu:$CIRCLE_PROJECT_REPONAME/production/.env`, unless `LUCOS_DEPLOY_ENV_BASE64` is set as a CircleCI project variable. The bypass exists in the orb for any project, but the sysadmin checked the CircleCI API and only `lucos_creds` actually has it set.
- **Builds:** `src/jobs/build.yml` calls `fetch-publish-creds` unconditionally, which scps `lucos_deploy_orb/publish/.env` from the same host. There is no bypass at all on this path.

`lucos_creds` runs on avalon. So for as long as avalon is down, **no project in the estate can build**, and only `lucos_creds` itself can deploy. The estate's ability to ship code — including any fix for the outage in progress — depends on the host that is down. It is a bootstrap circular dependency, and it appeared at the exact moment it was most expensive.

Two practical consequences:

- **The restore order is forced.** lucas42/lucos#296's original Step 3 sequence began with `lucos_configy` and `lucos_dns`, neither of which could deploy before `lucos_creds`. The approved sequence now starts with `lucos_creds`, restoring its store immediately after it boots so the window in which it serves fresh, wrong credentials stays short.
- **A workaround exists, and it is worth knowing.** CircleCI's selective job rerun redeploys a service from an already-published image, skipping the build step and its credential fetch entirely. That is how `lucos_root` was deployed for the test, reusing its last pre-incident pipeline.

**A second, independent orb defect blocks fresh builds for the same reason.** `src/commands/publish-docker.yml` probes the registry mirror with `MIRROR_HTTP_CODE=$(curl … -w '%{http_code}' … || echo "000")` and then tests `[ "$MIRROR_HTTP_CODE" != "000" ]`. When the connection is refused outright, curl prints its own `000` *and* exits non-zero, so the `|| echo "000"` fires too and the value becomes `000000`, which passes the `!= "000"` test. The probe therefore declares an unreachable mirror reachable, `Docker Login (mirror)` runs against it, and the build hard-fails instead of falling back to Docker Hub. `lucos_docker_mirror` is on avalon, so while avalon is down this fails every fresh build in the estate — `lucos_configy`'s at 23:49 on 2026-09-15 is a concrete instance. I've confirmed the probe code on `lucos_deploy_orb` `main`; the `000000` mechanism and its reproduction were worked out by lucos-system-administrator and are recorded on lucas42/lucos_deploy_orb#188, which already asks for the login step to fail open.

So a fresh build is blocked twice over while avalon is down: once by the publish credentials, once by the mirror probe. Both were worked around the same way, by rerunning a pre-incident pipeline and skipping the build.

lucas42 decided on 2026-09-15 that the credentials half stays as it is, and the answer is to write the bootstrap path down (lucas42/lucos#299). Giving other services their own copies of the credentials would create several sources for the same secrets, which drift; `lucos_creds` carries a bypass for its own bootstrap problem and nothing else needs one. The rule is that if `lucos_creds` is down it gets fixed first, before any other deploy — which is exactly what this rebuild did.

This is the same concentration risk as the disk, in a different layer: the estate's credential store, its container mirror, its CI dependency, its DNS primary, its monitoring and its alerting all live on one machine. The disk failure made all of them fail together.

### DNS: port 53 was already taken on the new host

`lucos_dns_bind` would not start on the rebuilt avalon: `failed to bind host port 0.0.0.0:53/tcp: address already in use`. **Confirmed, not inferred:** systemd-resolved's stub listener held port 53, and setting `DNSStubListener=no` freed it. This host is a stock Debian install in a way the old one had long since stopped being. Worth noting for anyone repeating it that lucas42's first pass at the change didn't take — the setting still read `yes` after the reboot — so the reboot alone didn't fix it; the change had to be reapplied. The symptom that showed first — a container with no network attachment, reporting healthy but unreachable — was a consequence of the failed start, so the obvious reading (a Docker networking fault) pointed away from the cause. It took dockerd's own log, and therefore lucas42's root access, to see it.

Two things make this worth recording rather than filing under bad luck:

- **The runbook never mentioned freeing port 53**, because the old host had been set up long before, by hand, and nobody had had to think about it since. lucas42's own notes carried a link about it that he hadn't followed yet. This is the class of step that only ever surfaces on a rebuild, which is exactly when nobody has done it for years.
- **`lucos_dns`'s compose file publishes the port on every interface.** I read it on `main`: `ports: ["53:53", "53:53/udp"]`. That is what produces the `0.0.0.0:53` in the error, and it is why the collision is with systemd-resolved's loopback stub listener. Publishing on avalon's public address instead would make the collision structurally impossible, and is an alternative to disabling the stub listener. Which of the two is right is a host-setup decision, not mine.

**The two DNS findings interact, and in the event were fixed together.** Disabling the stub listener is the usual way to free port 53 — but anything still pointing at `127.0.0.53` then has nothing to talk to, which is what the resolver timeouts looked like. Fixing either alone would have produced or preserved the other. lucas42 changed `resolv.conf` and disabled the stub listener in the same pass, which is the coherent combination. Whether the container resolver timeouts are gone is not something I can see from off-host; it wants confirming once containers on avalon resolve names again.

### Reboot: the host returns its services, and "healthy" doesn't mean reachable

The reboot to apply the `resolv.conf` change doubled as the first test of whether the rebuilt host recovers unaided. It does: every already-deployed container came back by itself and reported healthy, so `docker.service` is enabled at boot and the `restart: always` policies that every lucos compose file declares are doing their job. That is worth banking, with the caveat that it was tested with most of the estate still undeployed, so it proves the mechanism rather than the whole recovery.

The more interesting result is `lucos_dns_bind`. I'd predicted it wouldn't come back at all, on the grounds that a restart policy only revives a container that was running. That was wrong: the earlier CI deploy had created it, so Docker restarted it — **running, and reporting healthy, with no network attachment and no published ports**, because the port it needed was still held. A container can therefore pass its own healthcheck while being completely unreachable, which is the same lesson as the `/_info` boundary elsewhere in this estate: an internal check answers "is the process alive", not "can anyone reach it".

No new monitoring check is proposed for this. An unreachable service is exactly what monitoring's external `/_info` poll already catches; nothing flagged it here only because monitoring itself was still down — **which is not a reason the gap is benign, but another instance of the detector sharing a failure domain with the thing it detects** (lucas42/lucos#295). Adding a container-level "healthy but has no ports" check would duplicate, at some maintenance cost, a signal the estate already gets for free.

### Monitoring came back, detected everything correctly, and could tell nobody

When `lucos_monitoring` was redeployed it returned HTTP 500 on every endpoint for over an hour, while its poll loops ran perfectly well. Its container log gives the cause, and it is worth stating precisely because the intuitive explanation is wrong.

Every request is served by a synchronous `gen_server:call(StatePid, {fetch, all})` with Erlang's default **5-second** timeout, and the exception is that call timing out. But `{fetch, all}` does no network work: it builds the response from cached state. The call was waiting for the state server process to be *free*, and the process was blocked because **alert delivery runs in-band in it** — each alert was posting to loganne, which was still down, so each post waited out the router's 60-second timeout, and each email attempt retried against a refused port. The container sat at **0.01% CPU**: blocked, not busy.

So monitoring was simultaneously working and useless. It detected `lucos dns` and `lucos configy` failing, raised the alerts correctly, could deliver them on neither channel because both live on avalon, and meanwhile its own dashboard returned 500 to anyone asking what was going on. The alert-delivery half is the concrete instance now recorded on lucas42/lucos#295; the blocking half is lucas42/lucos_monitoring#312, with the principle that governs it on lucas42/lucos_monitoring#300, whose ADR already covers the same ground for a different mechanism.

It self-cleared at 01:14, when loganne alone came back and the API returned 200 with no restart — mail was still down, so the hanging channel, not the refusing one, was what starved reads. It is also, by construction, a fault that only appears during a serious outage: the worse the estate's health, the less usable its monitoring becomes.

### A check whose source disappears reads green, not unknown

Worth stating separately, because it is the least visible of the failures on this list and was only noticed because the rebuild made it happen in front of us.

While `lucos_schedule_tracker` was down, the systems whose only real check comes from it did not go red, and did not go unknown. **They went green**, and were counted among the healthy in the summary. `lucos_media_import` is the clearest case: with its scheduled-job check simply absent, the only check left was CircleCI, which was passing — so a system whose import job had not run since 2026-09-14 reported healthy throughout.

This is a different defect from the read-path blocking in lucas42/lucos_monitoring#312, and more dangerous than the other two "green means less than it looks" cases in this report. A container that is healthy with no network, or a backups dashboard that is green after a failed run, is at least *visibly* wrong to anyone who looks at the thing itself. A check that vanishes produces confident reassurance with no natural discovery path: nothing draws attention to a check that is no longer being made. The estate-wide healthy count is the number most likely to be glanced at during an incident, and it counts these systems as fine.

Filed as **lucas42/lucos_monitoring#313**. I had first routed this into lucas42/lucos_monitoring#300, and lucos-architect pushed back: that ticket's invariant is about unbounded or blocking work on the read path, and here the read path is fast, unblocked and working perfectly — it is the *answer* that is wrong. Different subject, different failure direction, different remedy. They were right, and an ADR that half-covered this would be worse than one that didn't mention it.

The mechanism, verified against the live API: a system entry carries `checks`, `metrics`, `name` and `status`, with **no declared or expected check set anywhere in the structure**. `checks` holds whatever sources reported, and `status` is computed over that set, so when a source vanishes its checks don't degrade — they cease to exist, and "all checks pass" becomes trivially true over the remainder. The `unknown` bucket is real and reachable by other paths, but a check with no source can never get there. **Absence isn't mis-reported; it's unrepresentable.** What that leaves is a decision rather than a bug: monitoring fails open, where this estate's auth consumers were deliberately made to fail closed.

### The repaired database was arithmetically wrong, and only one check could see it

The rescue-time repair of `media.sqlite` left `cum_weighting` — a running cumulative total used for weighted random track selection — inconsistent by **1,738.60**, against a 2026-09-13 backup that was exact to the last decimal. Only **5 rows** broke the invariant, and one of them held **1,738.50** of it: a 1,739-wide hole in the cumulative sequence before track 549. **It was not the two rows reinserted during the repair** — those and their neighbours account for about 10 — but the corruption the recovery was reconstructing from.

**What made it matter:** random selection draws a number up to `MAX(cum_weighting)` and takes the first track above it, so every draw landing in that hole returned the same track. **About 1 in 37 random picks — 2.7% — returned `M People - Moving on up`**, against a fair share of 0.0008%: over-selection by roughly **3,386×**. An invariant violation that reads as an arithmetic complaint was, in practice, one song playing every 37th track.

**I predicted it would self-heal and it did not.** I recorded that the weightings recalculation would repair it; that job then ran successfully and changed nothing, because `cum_weighting` is only ever maintained *incrementally* — adjusted by deltas on each weighting change or deletion — and **nothing in the estate rebuilds it from `SUM(weighting)`**. An inconsistency introduced from outside the app is therefore permanent. It was corrected on 2026-09-16 by a one-off recomputation with lucas42's approval, and the structural gap is lucas42/lucos_media_metadata_api#340.

### The mail outage was not what it looked like

For most of the rebuild `lucos_mail_smtp` crash-looped with `Fatal: master_service settings: Invalid settings: Unsupported dovecot_storage_version 2.4`, and the same error broke its CI. The striking part was that it appeared to be running an **unchanged, already-published pre-incident image** — which would have meant a container that had worked for weeks suddenly refusing to start with no input having changed.

**That reading was wrong, and the truth is more useful.** The container was not running the intended image at all. `lucos_deploy_orb` resolves the version to deploy from the **newest git tag rather than the checked-out commit** (lucas42/lucos_deploy_orb#193, raised Critical), so a *different* image — one built during the rebuild — was deployed in its place. That image carried an **unpinned dovecot**, and the newer package validates the setting more strictly: `2.4` is rejected where `2.4.0` is accepted.

Three separate faults stacked behind one symptom: a deploy selecting the wrong artefact, a dependency unpinned so its behaviour could change underneath a rebuild, and a configuration value that had always been marginal. The fix was a one-character change (lucas42/lucos_mail#80, merged 07:31, closing lucas42/lucos_mail#79), with pinning tracked as lucas42/lucos_mail#81 and the selection bug as lucas42/lucos_deploy_orb#193.

**Worth keeping:** "the image didn't change" is a claim about *which artefact is running*, not about the tag attached to it — and a rebuild, when everything is being re-tagged and re-deployed, is exactly when that assumption is least safe. It is settled by inspecting the running container, not the pipeline's intent.

### Photos: a whole day spent recovering data that was never lost

**This is the most important correction in this report, and the thing most worth learning from the incident.**

The version above — which is what the report said when it was merged — opened: *"`lucos_photos_photos` was the single volume deliberately excluded from backups, on the basis that it could be re-synced from the phone."*

**That was false.** `lucos_photos_photos` was backed up throughout. `lucos_configy`'s `config/volumes.yaml` gives it `recreate_effort: huge`, `backup_strategy: incremental`, and `skip_backup_on_hosts: [salvare, xwing]` — it is skipped **on the two hosts that don't hold it**, and backed up on avalon, where it lives. Read quickly, a `skip_backup_on_hosts` key looks like an exclusion. It is the opposite: it is what a volume that *is* backed up looks like when it only exists on one host.

aurora held a complete snapshot the entire time: **2,250 originals, 5,941 files, 11GB**, at `host/avalon/volume-snapshots/lucos_photos_photos/2026-09-14/`. Nobody looked, because everyone already knew there was nothing to find.

The cost of not looking:

- The runbook's plan to re-sync the library from lucas42's phone rested on a premise that was never checked against configy.
- A first, pre-fix resync ran and restored nothing — that part was a genuine bug (below).
- A second, post-fix resync transferred **2,253 files over 10.2 hours of wall-clock** to restore files that were already sitting on the NAS.
- Every downstream consequence in this report — the `cluster_faces` singleton persons, the stranded profile pictures, the database/volume mismatch that then needed its own restore — is a second-order effect of reprocessing a library that did not need reprocessing.

**The dedupe bug was real, and is worth keeping separate from the above.** With the files gone, `upload_photo()` looked a re-offered photo's `sha256_hash` up **in the database only** and returned `200 "already have it"` before writing anything. Observed rather than inferred: in one fifteen-minute window the API answered **619 `POST /photos` requests, every one a 200**, while the volume stayed at **4 files, 6MB**. Left alone, the resync would have reported ~2,252 items "already uploaded" and restored nothing. That is a genuine silent-loss defect, fixed by lucas42/lucos_photos#526 against lucas42/lucos_photos#525, both now merged and closed — and it would have bitten someone eventually. It simply wasn't on the path we needed.

**What to take from it.** The failure was not a missing backup, a broken tool or a bad decision under pressure. It was a **belief about the system that nobody checked against the system**, held confidently by everyone including this report, at a moment when checking cost one `grep` of a config file already checked out on disk. It survived being written into a runbook, into an incident report, into agent memory, and into a day of recovery work — because each of those inherited it from the last rather than from configy.

The generalisable rule: **during an incident, verify what you believe is *absent* at least as carefully as what you believe is present.** A false "we have that" gets caught the moment someone tries to use it. A false "we don't have that" is never caught at all, because nobody goes looking — it just quietly redirects the whole recovery down a more expensive path, and it looks like diligence the entire time.

### The restores that followed, and why a second one was needed

Once the snapshot was found, the photos volume was restored from it by rsync from aurora, completing at ~16:58 on 2026-09-16. That fixed the files and immediately exposed a subtler problem: **the database and the volume now came from different moments.** The database had been restored from the 2026-09-13 backup and had then been *running* all day — accepting 2,253 uploads, minting 769 new person records through face clustering, regenerating thousands of derivatives — while the files it was now paired with were a fixed snapshot. The result was a library that was internally inconsistent in three directions at once: three photo rows with no file, 769 persons whose `profile_photo_id` pointed at a derivative that no longer existed, and 732 person rows with no faces and no links.

lucas42's diagnosis is the sharpest reasoning of the incident and is worth quoting as the principle it is:

> "Surely the main cause of this issue has been doing processing on the database data between the restores of the two volumes. If both come from a backup around the same time, everything should align."

That is exactly right, and it generalises well beyond photos: **two volumes captured at the same moment are consistent by construction; no amount of repair work afterwards reproduces that property for free.** Every repair being contemplated at that point — nulling stranded `profile_photo_id`s, collecting orphaned persons, re-deriving thumbnails — was an attempt to reconstruct by inference something that a matched pair of backups provides by definition.

Two facts made it cleanly actionable:

- **The 2026-09-14 photos snapshot is byte-identical to the 2026-09-13 one** — not merely similar. The filenames *and inodes* match exactly in both `originals` and `derivatives`, because the rsync `--link-dest` snapshots hardlink unchanged files. So the restored files were precisely the state as of the 2026-09-13 backup run, and the matching database was that same run's tarball.
- **The rescued Postgres copy taken off the failing disk contained the same data.** Its newest activity was `2026-09-13 03:04:30Z`; nothing had happened in the service between then and the disk failure. So the two candidate databases were equivalent, and the choice could be made on principle — the nightly, being quiesced and under retention, rather than the rescue, which needed WAL recovery on start.

The database was restored from the 2026-09-13 nightly at ~22:20 on 2026-09-16. Predictions were recorded in advance and every one held exactly: eleven counted fields `2250|2139|111|931|1844|2113|0|0|0|2250|0`, curation unchanged at 127 confirmed faces and 66 contact-linked persons, Alembic already at head with no migration needed, and four consistency counts — rows with no file, files with no row, rows with no thumbnail, persons with no profile picture on disk — **all zero**.

### The three photos that really were unique

Three originals existed only on the live volume and in no backup: two taken on 2026-09-14 *after* the last snapshot, and one from 2026-07-25 that had never reached the server before. They were preserved because the photos restore moved the existing volume aside rather than overwriting it — a precaution worth naming, since it is the only reason this paragraph isn't about data loss.

They were returned through the ordinary upload endpoint from inside the API container, rather than by a second ten-hour phone resync. Two details made that the right call:

- **The server cannot short-circuit a duplicate.** The SHA-256 is computed from the request body *as it streams*, and the "already have it" check happens only after the whole file has arrived. A resync therefore re-transmits all 11GB even when the server already holds every byte. There is no cheap offer, and this is why "just re-sync" is never as free as it sounds.
- **The files carried their own dates.** All three held EXIF `DateTimeOriginal` with sub-second precision matching their recorded `taken_at` exactly, and the worker treats EXIF as authoritative. Chronology was preserved without anyone having to assert a date by hand.

All three returned 201, processed in under twenty seconds, and landed with `taken_at` correct to the millisecond. Integrity was confirmed by hashing the contents rather than comparing sizes — in this library the filename *is* the SHA-256, which makes the check self-verifying. Final state: **2,253 items, zero inconsistencies.**

### Restore: three snags worth knowing next time

All three came out of restoring `lucos_creds`, the first service back. They're reported by lucos-system-administrator; I haven't reproduced them myself, and they're recorded here because the next restore will meet them again.

- **`restore-volume.sh`'s `docker compose up --no-start` isn't safe on a multi-service compose file.** It's designed to recreate one volume's container, and a compose file with several services does more than that.
- **The rescue tarballs aren't shaped like the nightly ones.** They preserve the full original path inside the archive, so a restore has to move files up a level rather than unpacking in place.
- **`lucos_creds_ui` cached the wrong SSH host key.** It connected to the freshly-deployed backend before the restore, cached that identity, and then rejected the restored one. Removing its container cleared it. Anything that caches a peer's identity across a restore can do this.

### An 87-byte photos archive from March, and three wrong answers about it

While checking the restored backup set, an **87-byte `lucos_photos_photos.2026-03-06.tar.gz`** turned up on xwing, under `host/avalon/volume/`. An empty archive of the photo library, on a host configy says to skip, is an alarming-looking object, and it took three attempts to explain correctly. The answer is mundane; the three attempts are the interesting part.

**What it actually is**, from configy's history and the database:

| Date | Event |
|---|---|
| 2026-02-24 | `lucos_photos_photos` is added to configy (`5b7c41f`). The archive's internal directory mtime is 2026-02-24 17:13 — the volume's creation. |
| **2026-03-06** | A full backup run tars it to xwing. **The volume is empty**, so the archive is 87 bytes and contains exactly one entry, `./`. |
| **2026-03-09 14:06:49Z** | The first photo is uploaded — *three days after* the backup. |
| 2026-03-12 | Photos excluded from salvare (`c26d9c1`). |
| 2026-04-28 | Photos excluded from xwing, "now aurora is verified" (`f47bea4`). |
| 2026-06-10 | Photos opted into `backup_strategy: incremental` (`5212311`). |

So it is a **correct backup of a genuinely empty volume**, taken in the three-day window between the volume existing and the first photo arriving, and left behind on a host that stopped receiving photos backups seven weeks later. It is the only volume on xwing with a single fossil archive — every other volume there has 15 to 17 — so this is specific to photos rather than a general pattern.

**One thread is honestly not closed.** The xwing skip did not land until 2026-04-28, `2026-04-06` *is* a retained backup date on that host, and xwing has 88 GB free, so neither the config, retention nor capacity explains why no April archive of the growing library exists. salvare has never held one at all. Not worth chasing further — the data is on aurora and verified — but recorded as unexplained rather than smoothed over.

**The three wrong answers, which are the reason this section exists:**

1. *"A failed or truncated backup."* The intuitive reading of any near-empty archive, and wrong: `tar tzvf` shows a well-formed archive of an empty directory.
2. *"The volume is correctly skipped on that host."* Reassuring, plausible, and wrong — the skip postdates the file by seven weeks, and a skipped volume produces no archive at all rather than an empty one.
3. *"It's `lucos_media_import_state`, a self-deleting checkpoint whose empty archive is the success signal."* This one was mine, and it is the most instructive failure of the three. Searching for *"the 87-byte archive"* found a **different** file that happens to be 84–87 bytes too, and I explained the file I had found rather than the file I had been asked about. My first search for photos archives had used a glob that missed the two-level `host/avalon/volume/` path and returned nothing — and **I read that empty output as "no such file exists"** rather than as "my search didn't reach it", which is a rule I already had written down.

That third failure is the same shape as this incident's central lesson, in miniature: a confident negative that nobody checked. It is also a reminder that a coincidence of the specific search term — two unrelated files both 87 bytes — will happily produce a complete, internally consistent, entirely irrelevant explanation.

**Worth keeping regardless:** the reasoning in answer 3 is sound about the file it describes. A working-state file is not a history — checkpoints, cursors and lockfiles are typically created on failure and deleted on success, so their absence means things went well. For "did this job actually run?", the purpose-built sources are schedule-tracker and loganne, not the size of an artefact never meant to persist.

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

**Resolved 2026-09-16.** The rebuild and restore followed lucos-system-administrator's runbook, lucas42/lucos#296, using the data from the emergency-backups directory described above. 33 of avalon's 34 services are back and externally verified. The last of them, **`lucos_mail_smtp`**, was restored at **07:31 on 2026-09-16** (lucas42/lucos_mail#80, closing lucas42/lucos_mail#79) — see "The mail outage was not what it looked like" below. **All 55 monitored systems are now healthy**, and every CI check raised by the rebuild is cleared.

How it was rebuilt:

- Kimsufi replaced the failed disk, and lucas42 reinstalled avalon on Debian trixie 13.7 at the same IPv4 address, working from his own host-setup notes rather than the runbook's Step 1.
- The host serves fresh SSH host keys rather than the rescued ones, which is the fallback the runbook allows.
- The deploy pipeline is confirmed working, by deploying `lucos_root` and fetching its `/_info` — with a hand-injected placeholder envfile standing in for the credentials the pipeline could not fetch, so it demonstrates the pipeline rather than the pipeline's normal credential path.
- lucas42 has approved a restore sequence that starts with `lucos_creds`, for the reason set out under "Rebuild" above.
- `lucos_creds` is deployed and its store restored from the rescue tarball, and `lucos_configy` is deployed and verified externally. Both went via a selective rerun of a pre-incident pipeline, because fresh builds are blocked.
- `lucos_backups/init-host.sh` has run, so `/srv/backups` and the `lucos-backups` account exist. `lucos_docker_mirror` and DNS come next.

**Verification (step 5 of the runbook), carried out 2026-09-16 02:00–03:00:**

- **Every HTTP-serving system answers `/_info` externally.** All 41 configy-declared systems were swept from outside the estate, working from configy's own list rather than lucos_root's homepage — which lucos-architect found systematically under-reports which services are up, since its refresh sweep runs out of budget partway down the list (lucas42/lucos_root#158): the 31 that serve HTTP all returned 200, 28 of them fully clean. The other 10 have no HTTP surface — eight have no domain, and `lucos_dns` and `lucos_dns_secondary` have no `http_port`, so the TLS errors against their domains are just the router's default certificate on a name with no web vhost.
- **All five DNS zones match between primary and secondary**: `l42.eu` 1783445400, `s.l42.eu` 1780876665, `lukeblaney.co.uk` 20, `rowanblaney.co.uk` 17, `tfluke.uk` 25. That matters more than usual, because lucas42/lucos_dns#135 means the secondary holds no zone files on disk and the live sync is all there is.
- **The deploy pipelines that had failed only on their loganne step were re-run** — configy, docker_mirror, creds, dns and router — and all five reached terminal success. `lucos_firewall` was deliberately not re-run overnight: it reapplies rules on all three hosts, and with lucas42 away a bad rule would cost SSH access to whichever host it was applied to, against an upside of a green tick. It was re-run the following morning with him available. **A correction to my own reasoning at the time:** I justified the delay partly on agents' access "running through xwing", which is not true — `~/.ssh/config` has no `ProxyJump` or `ProxyCommand`, and avalon, xwing and salvare are each reached directly. Only aurora sits behind a gateway. So the exposure was per-host rather than one chokepoint. The decision to wait still stands on its own; the reason given for it was partly wrong.
- **A `create-backups` run was triggered end to end, and the first attempt failed.** Triggered 02:18:35Z, it reported an error: schedule-tracker showed `errors=1` and loganne recorded no completion event — while all sixteen `lucos_backups` checks stayed green. **Its cause is unknown and unrecovered**: the container drains cron output into a FIFO that is interleaved with the HTTP access log on stdout, and the traceback's body was lost. A second run, triggered 03:24:54Z with output captured to a file, **completed cleanly at 03:39:58Z** — 417 lines ending `Backups Complete`, zero error lines, `errors` back to 0, and loganne recording **"124 archives successfully backed up"**, the same archive count as every pre-incident run. The failure did not reproduce, so it is recorded here as an unexplained single-run transient rather than filed as a defect.
- **Restored data matches the figures recorded in the rescue README**, checked against the database engines rather than by inspection: contacts **30 tables**, eolas **41 tables**, photos **7 tables with the `vector` extension present**, media_metadata **14,755 tracks and 121,274 tags** with `integrity_check ok` and exactly the nine expected tables, and lucos_worlds' activity log latest at **2026-09-14 00:16:29** — lucas42's final edit, the one no backup contained. The aithne and creds stores could not be queried directly, as both run from images with no shell; they are covered indirectly by their services' own `db` checks, which are green.
- **aurora holds today's copies.** Reached the documented way, through the backups container's own Fabric path via the xwing gateway: **23 archives dated 2026-09-16** in `/share/backups/host/avalon/volume/`, host directories for avalon and xwing, and 1007 GB free at 73% used. After being unreachable and unchecked throughout the incident, the third copy is confirmed present.

**Final state, 2026-09-16 22:34 — all recovery complete.**

- **Photos: 2,253 items, fully self-consistent.** Every row has its file, every file has its row, every row has a thumbnail, and every one of the 931 persons has its profile picture on disk. Four independent consistency counts, all zero. Manual curation intact throughout: 127 confirmed faces, 66 contact-linked and 66 named persons, unchanged across both restores.
- **All data recovered.** Nothing was lost. The only permanent damage anywhere is the 14 unreadable sectors in `media.sqlite`, repaired to 2 rows restored from the previous day's backup.
- **Media weighting repaired** — `cum_weighting` drift 1,738.60 → 0, rows breaking the invariant 5 → 0, row count unchanged.
- **The 2025-01-06 yearly-retention set is back on avalon**, all nine files.
- **Mail delivering** since 07:31, so alerting is no longer blind.
- **55 of 55 monitored systems healthy**, and every CI check raised by the rebuild cleared.
- Both moved-aside volumes — `lucos_photos_photos_moved-aside-2026-09-16` and `lucos_photos_postgres_data_moved-aside-2026-09-16` — are retained deliberately, pending lucas42's decision. They are the rollback path for the two restores, and the second of them is the only reason the three unique originals survived.

**Still open after the rebuild (superseded — recorded as it stood on the morning of 2026-09-16):** **the media queue came back empty** — `lucos_media_manager`'s state was restored from the 2026-09-13 backup, which recorded it as playing with nothing queued, so there is nothing to play until something repopulates it. If anyone reports that their music has stopped in the next few days, that is why, and it is also what keeps lucas42/lucos_media_linuxplayer#146 crash-looping. **The photo originals were believed still missing at this point** — they were not; see "Photos" and the final state above. Everything else that was open the morning after has since closed: `lucos_mail_smtp` was fixed at 07:31, the media `weighting` inconsistency was corrected, the deliberately-held CI re-runs all went green, and `lucos_media_manager`'s deploy failure turned out to be a start-up race rather than a bad deploy — the container it created is the one running, on the image that workflow built, and `docker compose up --wait` simply gave up 18.2s in while the healthcheck was still settling. Verification must include a **triggered `create-backups` run**, not just green `/_info`s, because backups is a cron path that a green `/_info` cannot exercise.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| OVH/Kimsufi disk replacement — replaced 2026-09-15 | external support ticket (lucas42) | Done |
| Rebuild avalon and restore from the emergency backups — 33 of 34 services restored and externally verified by 2026-09-16 03:39; `lucos_mail_smtp` is the exception, on lucas42/lucos_mail#79 | lucas42/lucos#296 (runbook) | Done |
| Decide the rebuilt avalon's disk layout (RAID or a second disk?) — decided: **stays single-disk for now**, as the server is mid-way through a year-long contract; revisit at renewal. This raises the value of lucas42/lucos_docker_health#118 | lucas42/lucos#296 (runbook, item 2) | Done |
| Delete the rescued SSH host keys (`rescue/avalon-ssh-host-keys/`), and the rest of the emergency-backups directory, from xwing and salvare — deferred by lucas42: nothing is deleted until everything is restored and working, and it stays a while beyond that. The rebuilt host generated fresh keys, so these were never installed | lucas42/lucos#304 | Open |
| Correct the emergency-backups README, which named the rescued host-key fingerprint as the one to expect — corrected 2026-09-16 01:01, verified against the file on xwing. It now records that the reinstall took the documented fallback and that avalon serves freshly generated keys, with the new fingerprint confirmed three ways | lucas42/lucos#296 | Done |
| Clear avalon's old host key wherever a `known_hosts` still holds it — the sysadmin found no `lucos-agent` entries on xwing or salvare; `~lucos-backups/.ssh/known_hosts` needs root to check | lucas42/lucos#303 | Open |
| Rotate credentials possibly exposed on the departing disk: the lucos_creds `server_key` and the aithne credential store. avalon's OS-level host keys are moot, as the rebuild generated fresh ones — blocked on lucas42/lucos#296 | lucas42/lucos#298 | Blocked |
| Cost the recovery path to aurora, the off-host backup destination: its network route is via xwing and was open throughout, but the only credential that authenticates lives in the `lucos_backups` container on avalon — same shape as lucas42/lucos#299. Not data loss, since the data is on aurora regardless; unmeasured recovery time | lucas42/lucos#301 | Open |
| Document the bootstrap path for CI being unable to build or deploy while `creds.l42.eu` is down — decided 2026-09-15, documentation only: lucas42 rejected every option that would give another service its own copy of the credentials, because multiple sources drift. `lucos_creds` is fixed first, then everything else. Ready, owner SRE | lucas42/lucos#299 | Open |
| Make the mirror login fail open, and stop the mirror probe reading a refused connection as reachable — the `000000` defect found during this rebuild is recorded there | lucas42/lucos_deploy_orb#188 | Open |
| Reconcile lucas42's own host-setup notes with lucas42/lucos#296's Step 1 into one runbook, marking which steps are his and which the agents'. Freeing port 53 belongs in it, and so does the question of whether `lucos_dns` should publish on the public address rather than every interface — suggested, after the rebuild | lucas42/lucos#300 | Open |
| Establish why dockerd's resolver times out against `127.0.0.53` on the rebuilt host — lucos-system-administrator is preparing a fix for lucas42 to run; may share a cause with the port 53 collision | lucas42/lucos#302 | Open |
| Restore avalon's swapfile to its previous size (~512 MB now, against roughly 4.5 GB before) — flagged by the sysadmin; `lucos_photos_worker` and `redis` were the known memory consumers. The runbook's swapfile step gave no size, which is the likely reason for the gap, and now specifies one, so a future rebuild shouldn't reproduce it | lucas42/lucos#303 | Open |
| Fix the DNS secondary on xwing, which cannot write zone files to disk and served all five zones from memory throughout the outage — filed 2026-09-16. Failing continuously since at least 26 August; root cause not established, and the four obvious explanations are ruled out in the issue | lucas42/lucos_dns#135 | Open |
| Stop `lucos_media_linuxplayer` crash-looping when the manager reports playing with an empty queue — ready/High, owner developer. Raised High for masking salvare's container-health check, not for the player itself | lucas42/lucos_media_linuxplayer#146 | Open |
| Restore production SMTP — **done 2026-09-16 07:31** (lucas42/lucos_mail#80, closing lucas42/lucos_mail#79). It was not an unchanged image: the orb deployed a different one built during the rebuild, carrying an unpinned dovecot whose validator rejects `2.4` for `2.4.0` | lucas42/lucos_mail#79 | Done |
| Pin the dovecot package version so a rebuild can't pick up a stricter validator | lucas42/lucos_mail#81 | Open |
| Fix deploy version resolution, which picks the newest git tag rather than the checked-out commit — it deployed the wrong artefact and caused the mail outage | lucas42/lucos_deploy_orb#193 | Open — Critical |
| Recover the photo originals — **done 2026-09-16 22:34**, and not by the route this report first described. They were never lost: aurora held a complete snapshot throughout. Restored from it, the database matched to it from the 2026-09-13 nightly, and the three genuinely-unique originals returned through the upload endpoint. Final state 2,253 items, four consistency counts all zero | lucas42/lucos_photos#525 | Done — fix merged as lucas42/lucos_photos#526 |
| Reject zero-byte uploads: a zero-byte video is accepted and stored, and the empty-file hash is a constant, so it is a single global slot every future zero-byte upload collides with | lucas42/lucos_photos#527 | Open |
| Stop `sweep-chronically-stuck-count` counting healthy backlog as stuck, and stop the per-item warning flooding the log — 116,633 `chronically stuck` warnings in 6.5 hours, 62.5% of the worker's entire log, for 2,182 media items and 769 persons that were merely queued | lucas42/lucos_photos#528 | Open |
| Make worker stalls diagnosable — a 12-minute silent stall could not be investigated at all: no `faulthandler`, no `py-spy` on avalon, no stack trace obtainable. Three lines to fix | lucas42/lucos_photos#529 | Open |
| Collect emptied person rows: the policy exists but is wired into the merge helper only, so `assign_person` and `unassign_person` strand orphans. 732 in production before the database restore cleared them; the defect remains and they will re-accumulate | lucas42/lucos_photos#530 | Open |
| Key the profile-picture backstop on the file rather than the database column — `_enqueue_missing_profile_pictures` filters on `profile_photo_id IS NULL`, so a person whose derivative is missing can never self-heal. Same "DB says yes, disk says no" class as lucas42/lucos_photos#525 | lucas42/lucos_photos#531 | Open |
| Document that `*-v4.s.l42.eu` names are port-specific deploy routes, not host aliases — they share one NAT address where SSH answers as xwing, so `ssh salvare-v4` silently lands on the wrong host, and anything IPv4-only cannot reach salvare by name at all | lucas42/lucos_dns#136 | Open |
| Decide the fate of the two moved-aside volumes, `lucos_photos_photos_moved-aside-2026-09-16` and `lucos_photos_postgres_data_moved-aside-2026-09-16` — retained as the rollback path for the two restores. Neither is declared in configy, so neither is backed up; that is acceptable only while they are redundant copies, which they now are | lucas42/lucos#304 | Open — decision needed |
| Restore the 2025-01-06 yearly-retention set to avalon | lucas42/lucos#296 | Done — all nine files present in `/srv/backups/local/volume/` |
| Correct the media `weighting` inconsistency left by the repaired database — **done 2026-09-16**, approved by lucas42: `cum_weighting` recomputed as the running sum in existing order, drift **1,738.60 → 0**, rows breaking the invariant **5 → 0**, row count unchanged; pre-change copy on xwing. **My earlier reading that `all-tracks` would repair it was wrong** — that job ran successfully and changed nothing, because `cum_weighting` is only ever maintained incrementally | lucas42/lucos#302 | Done |
| Give the estate a way to rebuild `cum_weighting`, since nothing can — so any restore or interrupted write leaves it permanently skewed, detectable only by one check | lucas42/lucos_media_metadata_api#340 | Open |
| Confirm Let's Encrypt renewal works on the rebuilt host before 2026-10-18, when the restored certificates expire. Both paths run inside the router container: `update-domains.sh`'s per-domain `certbot certonly`, at startup and daily at 22:16, and the stock Debian `certbot renew` cron. Owner: sysadmin — the first natural attempt falls around 2026-09-18, 30 days before expiry, so it tests itself within days | lucas42/lucos#302 | Open |
| Build tooling to rotate the lucos_creds master `data_key` | lucas42/lucos_creds#565 | Open |
| Make `create-backups` run overnight as designed | lucas42/lucos_backups#415 | Open |
| Decide whether alerting should survive avalon going down hard — decision a concrete instance from this rebuild is now recorded on the ticket: alerts correctly raised for `lucos dns` and `lucos configy`, deliverable on neither channel | lucas42/lucos#295 | Decision needed |
| Stop monitoring's alert delivery blocking its read path, so the dashboard stays usable when the estate is broken — filed 2026-09-16 at lucas42's request. Only bites when loganne or mail are themselves down, which is rare, but that is the incident case | lucas42/lucos_monitoring#312, with the governing principle on lucas42/lucos_monitoring#300 (ADR) | Open |
| Add a host disk-health signal (I/O error rate, optionally SMART) — decision | lucas42/lucos_docker_health#118 | Decision needed |
| Record this occurrence against the alert-to-action gap | lucas42/lucos#290 (comment) | Done |
| DNS zone expiry deadline (2026-10-12 07:09:51 UTC) — no longer live: the secondary re-established contact with the rebuilt primary at 00:37 on 2026-09-16 and all five serials match | recorded on lucas42/lucos#294; no issue, by lucas42's decision | Done |
| Build emergency copy lists from configy; verify copies as data | lucas42/lucos_claude_config `0406391` | Done |
| Run long production operations under Monitor, not in the foreground | lucas42/lucos_claude_config `f431318` | Done |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] Yes — see note below.

The rescued data includes the **credential store** (`lucos_creds_store`, including its key files), the **aithne credential store**, and host account `authorized_keys` files. They're stored only in `~lucos-agent/emergency-backups-2026-09-14/` on xwing and salvare, where both the home directory and the folder are mode 700. **Those permission bits are not the real boundary**, as lucos-security points out: `lucos-agent` already has root-equivalent access on both hosts through `docker` group membership (lucas42/lucos_agent_coding_sandbox#102), so anyone with that SSH access can read these files whatever the mode says. This is the standing access agents have to production credentials day to day, not new exposure created by the rescue — but the containment should not be read as file permissions. avalon's **SSH host keys** (private) were also copied, on 2026-09-15 at lucas42's request, into `rescue/avalon-ssh-host-keys/` in the same folder, with files at 600. On 2026-09-15 lucas42 decided, on lucos-architect's advice, that the rebuild would keep the hostname `avalon` and reuse the ed25519, ecdsa and rsa pairs (lucas42/lucos#296). **In the event, the rebuilt host generated fresh keys instead** — the fallback that runbook allows — so the rescued private keys were never reinstalled. They are therefore now the only copies of keys belonging to a host that no longer exists. **They stay where they are for now.** lucas42's rule for the rebuild is that nothing is deleted from the emergency-backups directory until everything is restored and working, and for a while after that: deletion is one-way while a restore is still in flight, and it buys nothing here, since the same mode-700 directory already holds the rescued credential store — anyone able to read the host keys has more than the host keys. Two consequences follow: the emergency-backups README still names the rescued fingerprint as the one to expect and needs correcting, and anything holding avalon's old host key in a `known_hosts` file will refuse to connect until that entry is cleared. `/etc/shadow` and all other private keys were deliberately **not** copied. No credential values appear in this report or in the linked issues.
