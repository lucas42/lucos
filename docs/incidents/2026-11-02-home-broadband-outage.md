# Incident: Home broadband outage — 33h of hard-down home hosts, false flapping on avalon, and IPv6 that didn't come back

| Field | Value |
|---|---|
| **Date** | 2026-11-02 to 2026-11-03 |
| **Duration** | ~33h (2026-11-02 ~01:05 UTC to 2026-11-03 10:11 UTC). Plus ~13.5h more for salvare alone, which stayed unreachable from avalon until 23:43 UTC because IPv6 didn't come back. |
| **Severity** | Partial degradation (everything hosted at home was unreachable), plus a data-risk window: no off-avalon backup copies for ~47.5h |
| **Services affected** | Unreachable: every system on xwing or salvare (`lucos_private`, `lucos_static_media`, `lucos_dns_secondary`, `lucos_media_import`, `lucos_media_linuxplayer`, and the home-side halves of `lucos_router`, `lucos_firewall` and `lucos_docker_health`). Side-effects on avalon: `lucos_backups` (false alerts, failed backup runs) and `lucos_docker_health` (false `avalon` alerts). |
| **Detected by** | Monitoring alerts. The flapping was raised by lucas42, who asked why a network that was hard down the whole time produced flapping alerts. |

---

## Summary

lucas42's home broadband was hard down for about 33 hours. Every lucos host at home (xwing, salvare, and the NAS aurora) was unreachable from avalon for the whole period. lucas42 has accepted that services hosted at home are unavailable when the home connection is down (in chat, relayed by team-lead on 2026-11-03), so this report doesn't treat that as the problem. It covers three side-effects:

1. **A hard-down network produced hourly *flapping* alerts on two avalon services.** `lucos_backups` and `lucos_docker_health` each went red and recovered every hour. The hourly cycle accounts for 116 of the 137 alerts those two systems sent during the outage. The cause was `lucos_backups`' single-threaded web server, which blocked for about 7 minutes every hour waiting on SSH SYN timeouts to the unreachable hosts (lucas42/lucos_backups#374).
2. **No backup left avalon for about 47.5 hours.** Every off-avalon backup destination (aurora, xwing, salvare) is at home. When the first post-outage run did reach the home hosts, a separate bug meant repository archives skipped xwing (lucas42/lucos_backups#460).
3. **IPv6 didn't come back when the ISP restored service.** The home router advertised no IPv6 prefix, so salvare, which avalon reaches over IPv6, stayed unreachable for another ~13.5 hours. IPv6 came back within two minutes of lucas42 restarting the router (lucas42/lucos#331).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-11-01 15:58 | Last successful `create-backups` run before the outage: 127 archives, copied to the home hosts. |
| 2026-11-02 ~01:05 | Home broadband goes down (by 01:07 at the latest; see the next row). The first alerts arrive from 01:08 over the next 5 minutes, all for home-hosted services or checks that probe them: `lucos_time` `media` (01:08:59), `lucos_media_import` (01:09:30), `lucos_docker_health` `salvare-v4`/`xwing-v4` (01:10:30), `lucos_dns_secondary` `port-53-reachable` (01:11:32), `lucos_private` and `lucos_static_media` (01:12), `xwing` (01:13:01). |
| 2026-11-02 01:12 | The first false avalon-side alerts, `lucos_backups` `fetch-info` and `lucos_docker_health` `avalon`, are already firing, triggered by the 01:07 refresh. So the connection had already dropped by 01:07, and the first alerts lag the real start by a few minutes. |
| 2026-11-02 02:11–02:15 | First full instance of the hourly cycle: `lucos_backups` `fetch-info` and `lucos_docker_health` `avalon` go red at about :11–:12 and clear at about :13–:15. The cycle recurs in 29 of the ~33 hours of the outage. |
| 2026-11-02 03:25 | `create-backups` slot: expected to no-op, because the last success was 11.5h earlier, inside the 20h skip-if-fresh window. Logs for this period are lost. |
| 2026-11-02 15:25 | `create-backups` starts with every home host blackholed. Inferred from schedule-tracker counters (logs from this period were lost in a 11-03 07:17 restart): it was **still running at the 11-03 03:25 slot**, which therefore no-op'd on the lock, and it finished with an error some time before 07:17. That's a run of roughly 12–16 hours. See Analysis. |
| 2026-11-03 04:55 | `prune-backups` fails for all three home hosts ("host unreachable"). |
| 2026-11-03 10:11 | ISP restores service. `xwing` recovers at 10:11:31; xwing and aurora are reachable again. salvare is not. |
| 2026-11-03 10:13 | Last avalon-side flap alert. |
| 2026-11-03 10:35 onwards | Tracking runs after recovery take 10–15s: salvare now fails *fast* ("Unable to connect", not a SYN timeout), so a refresh no longer blocks the server for minutes. No further flapping. (The 11:07 hourly refresh itself got a cooldown 429 and didn't run; the evidence comes from the in-process runs, which use the same code.) |
| 2026-11-03 10:39 | lucos-system-administrator opens lucas42/lucos#331: neither xwing nor salvare has a global IPv6 address, and salvare is unreachable from avalon. |
| 2026-11-03 ~10:40 | lucos-site-reliability traces the flapping to `lucos_backups`' hourly `/refresh-tracking` blocking its single-threaded server, and adds the evidence to the existing lucas42/lucos_backups#374 (later raised to High priority). |
| 2026-11-03 15:25 | `create-backups` runs. Every volume and one-off file is archived and copied to aurora and xwing, the first off-avalon copies since 11-01 15:58. Copies to salvare fail. Repository archives reach avalon only: 98 to avalon, 98 failed to salvare, 0 to xwing. aurora never takes repository archives, by design (lucas42/lucos_backups#460). |
| 2026-11-03 22:50–22:53 | While investigating that run, lucos-site-reliability reads `/var/log/cron.log` inside the container, not knowing it is a FIFO, and briefly competes with its drain. No cron output is written in that window, so nothing is believed lost (lucas42/lucos_backups#461). |
| 2026-11-03 23:37 | lucos-system-administrator root-causes the IPv6 loss: the home router is sending router advertisements with **no prefix and a router lifetime of 0**. |
| 2026-11-03 ~23:40 | lucas42 restarts the home router (time from his message, relayed by team-lead). |
| 2026-11-03 23:42 | IPv6 is back on xwing and salvare, with the same prefix and the same public IPv4, so no DNS or `hosts.yaml` changes are needed. |
| 2026-11-03 23:43 | `lucos_backups` tracking reaches salvare again. Source: the in-process run finished at 23:43:22 with no salvare error; monitoring's 23:43:39 alert no longer lists `host-tracking-failures`; `/_info` at 23:46 shows `host-count` 4 and the check ok. |
| 2026-11-04 03:02 | lucos_agent's scheduled `prune-images` succeeds on all three hosts, avalon, xwing and salvare (pipeline 122). |
| 2026-11-04 03:25–03:57 | `create-backups` succeeds: 126 archives. Volume and one-off copies reach aurora, salvare and xwing, and repository archives reach avalon, salvare and xwing (98 each). The `create-backups` check clears. |
| 2026-11-04 04:49–05:06 | `prune-backups` succeeds ("Pruning Complete") and its check clears. Monitoring shows 56/56 systems healthy. **Incident fully resolved.** |

---

## Analysis

> **Note on cross-repo references:** All incident reports live in the `lucos` repo. Every reference below is fully qualified.

### The outage itself (accepted, not analysed)

The ISP connection dropped and came back about 33 hours later. lucas42 has accepted that services hosted at home are unavailable while the home connection is down, so this report deliberately makes no redundancy or availability recommendations for xwing or salvare. Everything below is about how the rest of the estate *behaved* during and after the outage. (Scope set by lucas42 in chat, relayed by team-lead on 2026-11-03.)

### Why a hard-down network produced flapping alerts

The xwing and salvare checks behaved correctly: they went red at about 01:10 and stayed red until 10:11. The flapping came from two avalon-side checks with one shared cause.

- `lucos_backups`' cron fires `POST /refresh-tracking` at :07 every hour. That handler runs `fetchAllInfo()` synchronously inside a **single-threaded** `HTTPServer`, so nothing else is served until it finishes (lucas42/lucos_backups#374, open since July).
- Tracking SSHes to each backup host in turn, with **no connect timeout** configured. To a blackholed host, each connection waits out the kernel's SYN retries (`tcp_syn_retries=6`, about 127s), each failing with `[Errno 110] Operation timed out`. With three home hosts unreachable, the server was blocked for **~6m48s every hour**: `POST /refresh-tracking` returned at 08:13:48 and 09:13:48 on 11-03, against about 20s normally.
- That is long enough to trip two independent checks:
  - lucos_monitoring's `fetch-info` on `lucos_backups` went red at about :11–:12, and cleared at about :13–:14 once the queued requests drained.
  - The container's own Docker healthcheck (`wget /_info`, interval 10s, timeout 5s, retries 3) failed, so Docker marked `lucos_backups` unhealthy. `lucos_docker_health` on avalon correctly reported that as `Unhealthy containers: lucos_backups` every minute from :07:50 to :12:49, which turned its `avalon` check red.
- Each change in a system's set of failing checks generates a fresh alert, so each cycle produced four alert emails: one when each system went red and one when each recovered. That's 4 × 29 = 116 alerts. The other 21 of the 137 on these two systems were the initial alerts as the outage began, alerts when new steady failures were added (`prune-backups`, deploy restarts), and the unrelated estate-wide `circleci` alert at 23:39. This is the alert-storm class that lucas42/lucos_monitoring#302 exists to coalesce.
- Beyond the noise, there was a masking risk: with two avalon checks red and recovering every hour on top of the steady home-host reds, a genuine avalon alert would have been easy to miss.

**Why it wasn't strictly hourly.** In 4 of the ~33 hours the cycle didn't fire. This was observed directly after the outage: the in-process retry timer (every 5 minutes, in its own thread, which doesn't block the server) occasionally finished within the 60s before :07. The cron's refresh was then rejected by the cooldown with a 429, so that hour's blocking run never happened.

**Why it stopped before IPv6 returned.** Once the ISP was back, salvare's address was no longer blackholed. It was simply unroutable, so connections failed immediately ("Unable to connect to port 22") instead of waiting out SYN retries. From the first post-recovery tracking run (10:35) onwards, tracking took 10–15s and nothing alerted, even though salvare was still unreachable.

lucas42/lucos_backups#374's original impact statement said this blocking "doesn't alert" at its normal ~20s. This incident is the latent case that ticket warned about, triggered by unreachable hosts rather than a slow GitHub. The fix (`ThreadingHTTPServer`) is unchanged and would have prevented all 116 alerts from the hourly cycle.

### No backup left avalon for ~47.5 hours

`lucos_backups`' off-avalon storage destinations are aurora, xwing and salvare, all at home. From the last pre-outage success (11-01 15:58) to the first post-outage run that reached a home host (11-03 15:25), new archives existed only on avalon. For that window, avalon's single disk held the only copy of everything changed since 11-01 15:58. That same single disk failed on 2026-09-14 (see the avalon disk failure report). Nothing was lost, and this is recorded as impact, not as a recommendation. lucas42 has confirmed that backup-destination diversity is out of scope: the existing off-avalon backups were never at risk, and with no broadband he wasn't making important changes on avalon (in chat, relayed by team-lead, 2026-11-04).

### A `create-backups` run took roughly 12–16 hours and swallowed the next slot

There's no connect timeout anywhere in `lucos_backups`' SSH/scp paths. The `timeout=` arguments in `src/` bound how long each command *runs*, not how long a connection attempt takes. So every copy to a blackholed host waits out the kernel's SYN retries, about 127s each. With all three destinations blackholed and ~81 items, that adds up to hours.

The run logs from this period were lost in the 11-03 07:17 container restart. Schedule-tracker's counters still pin it down, though. At 15:47 on 11-03, `create-backups` showed exactly **2** consecutive errors since its last success, and the check first went red at 15:47:51 on 11-03, not earlier. The only sequence I can find that fits both is:
- the 11-02 15:25 run was **still holding the lock at the 11-03 03:25 slot**, which therefore exited through the "previous run still in flight" path and reported success;
- that 15:25 run then finished with an error before the 07:17 restart;
- the 11-03 15:25 run was the second error.

That's an inference from counters, not a direct observation, but it gives a run of roughly 12–16 hours.

The *outcome* wasn't changed by it: with every destination unreachable, the 03:25 slot would have failed anyway, and local archives on avalon were still created. That's why the connect timeout stays in "considered and not filed" below. It would make a doomed run fail sooner, but it wouldn't have saved any copy.

### Repository archives skipped xwing in the first post-outage run

`create-backups` copies volumes and one-off files through `backupToAll`, which tries every host and then raises once at the end. Repository archives go through `Repository.backup()` instead, whose per-host loop has `try/finally` but no `except`. The first host that fails ends the loop. In the 11-03 15:25 run the loop reached salvare before xwing. salvare failed, so **xwing received 0 of 98 repository archives** even though it was reachable, and new repository archives existed on avalon only (aurora never takes them, by design). Volumes and one-off files in the same run were copied to aurora and xwing as designed. GitHub holds the primary copy of every repository, so the practical risk is small, but the gap appears exactly when one storage host is already down (lucas42/lucos_backups#460).

### IPv6 didn't come back with the ISP

When the ISP restored service at 10:11, IPv4 came back but IPv6 didn't. lucos-system-administrator found (lucas42/lucos#331) that the home router was sending router advertisements with no prefix and a router lifetime of 0. That is consistent with the router not having re-acquired its IPv6 prefix delegation from the ISP (the sysadmin's best reading; nobody can see inside the router, so it's not confirmed). Either way, the hosts had nothing to build a global address from. Two early leads were red herrings:
- `accept_ra` reading 0 on both hosts. NetworkManager appears to handle router advertisements itself on these hosts: it held the router's DNS entry while the kernel's `accept_ra` read 0. Nobody has read NM's source or logs to confirm this.
- salvare's `eth0` being down. It has had no carrier since boot on 2026-09-27; salvare runs on `wlan0`.

lucas42 restarted the router at about 23:40 (time from his message, relayed by team-lead). By 23:42 both hosts had the same prefix and a default route again. The restart is the likely cause, but self-recovery can't be ruled out from what was observed. One supporting fact: an agent sandbox whose traffic leaves through the home connection had requests reach avalon at 23:40:35 and 23:42:45 but lost one in between, which fits a router restart in that window. A similar failure happened on 2026-04-19 (from lucos-system-administrator's notes): the router stopped sending router advertisements entirely, so the old prefix aged out. That's a different symptom with the same suspected cause (the router losing its prefix delegation), and the same remedy (a router restart).

Because avalon reaches salvare at `salvare.s.l42.eu`, which has only an AAAA record, salvare stayed unreachable for backups and tracking for ~13.5 hours after the ISP outage itself had ended. Once the ISP was back, it *was* reachable over IPv4 at `salvare-v4.s.l42.eu` through a router port forward (one SSH keyscan by the sysadmin, between the ISP's return and the router restart). Nothing in the estate uses that path for backups.

### Other side-effects

- **lucos_agent's `prune-images` failed silently.** lucos_agent's scheduled `prune-images` job failed for xwing and salvare at 03:02 on both 11-02 and 11-03, at the `ssh-keyscan` step. The outage window fits; lucos-system-administrator spotted it but couldn't confirm the cause from the step output, which was empty. It cleared at the next scheduled run: 03:02 on 11-04 succeeded on all three hosts. Nothing alerted, because monitoring's `circleci` check ignores repos listed only in `scripts.yaml` (lucas42/lucos_monitoring#321, existing).
- **Not related, but it overlaps in time:** at about 23:35 on 11-02, mid-outage, the `circleci` check went red on 52 of 56 systems. That's a separate failure: lucos_monitoring had no CircleCI token, and the CircleCI API stopped serving the unauthenticated requests it had been making (lucas42/lucos_monitoring#320). Anyone scanning alert history for this period should not link the two.

### A side-effect of the investigation: the cron.log FIFO

`lucos_backups` routes all cron output through `/var/log/cron.log`, a named pipe that `startup.sh` drains to the container's stdout. Reading it directly (`grep` via `docker exec`) consumes whatever it reads, and the read blocks indefinitely. During the investigation a `grep` ran against it for about 3 minutes before being killed by PID. No cron job writes in that window, so it is not believed to have lost anything. But nothing in the repo warns that the path isn't a file (lucas42/lucos_backups#461).

---

## What Was Tried That Didn't Work

- **`sysctl -w net.ipv6.conf.eth0.accept_ra=2`** (proposed in the first version of lucas42/lucos#331, never run). `accept_ra` = 0 appears to be NetworkManager's normal arrangement on these hosts, not a fault. The real fault was upstream, at the router.
- **"Salvare is IPv6-only, so it is completely unreachable"** (the first version of lucas42/lucos#331). Once the ISP was back, it was reachable over IPv4 on `salvare-v4.s.l42.eu:2202`. Only the IPv6 name that `lucos_backups` uses was unreachable.
- **Using the 11:07 refresh on 11-03 as the test that fast failures no longer block the server.** That refresh was rejected by the cooldown (429), so it didn't exercise the blocking path. The conclusion was instead verified from the 10–15s duration of every in-process tracking run after recovery, which use the same code path.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Serve `lucos_backups` HTTP with `ThreadingHTTPServer` so `/refresh-tracking` can't block `/_info` and the healthcheck (the cause of the 116 hourly-cycle alerts) | lucas42/lucos_backups#374 | Open (Priority raised to High) |
| `Repository.backup()`: carry on to the remaining hosts when one fails, then raise once | lucas42/lucos_backups#460 | Open |
| Warn that `lucos_backups`' `/var/log/cron.log` is a FIFO that reading consumes | lucas42/lucos_backups#461 | Open |
| Restore IPv6 on xwing and salvare after the outage | lucas42/lucos#331 | Done (IPv6 back by 23:42, after a router restart at ~23:40) |
| *Related, existing:* the outage made lucos_agent's scheduled `prune-images` fail and nothing alerted, because monitoring's `circleci` check ignores `scripts.yaml` repos | lucas42/lucos_monitoring#321 | Open |
| *Related, existing:* coalesce alerts that share a cause. The 137 alerts are the class this addresses; lucas42/lucos_backups#374 removes this particular trigger. | lucas42/lucos_monitoring#302 | Open |
| *Considered and not filed:* an SSH/scp connect timeout in `lucos_backups`, and making the "previous run still in flight" no-op report something other than success. For tracking, a connect timeout prevents no alert once lucas42/lucos_backups#374 lands. For `create-backups`, it would shorten a run like the inferred 12–16h one above, but every destination was unreachable, so the run and the slot it swallowed would both have failed anyway. **Not harmless in general, though (lucos-architect):** a swallowed slot ticks schedule-tracker green, so a wedged run reads as healthy until it finishes. The case where that matters is a *partial* blackhole, where copies to the healthy hosts are delayed. That's my estimate, not measured: roughly 127s per item for one blackholed host, about 3h of delay across ~81 items, plus about 3.5h of repository connects. Not filed on proportion: nothing is lost, the run does finish and reports its own error, which turns the check red, and with one host blackholed the estimated ~6.5h stays inside the 12h slot gap, so no slot is swallowed. | n/a | Not filed |
| *Considered and not filed:* anything about redundancy or availability for xwing/salvare, including detecting or automating recovery when the router fails to restore IPv6 after an outage (a similar failure happened on 2026-04-19). | n/a | Out of scope by lucas42's decision (in chat, relayed by team-lead, 2026-11-03) |
| *Considered and not filed:* backup **failure-domain diversity** for avalon's data. All three off-avalon destinations share one failure domain, the home broadband (raised by lucos-architect). lucas42 is fine that new changes on avalon weren't copied off-host during the outage: the existing backups on xwing, salvare and aurora were never at risk, and with no broadband he wasn't making important changes to systems on avalon. | n/a | Out of scope by lucas42's decision (in chat, relayed by team-lead, 2026-11-04) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

Personal data, not credentials: the backup-run logs and monitoring `debug` strings consulted for this report list lucas42's personal one-off backup filenames. None are quoted here; counts are used instead. No credentials or security-relevant details were involved.
