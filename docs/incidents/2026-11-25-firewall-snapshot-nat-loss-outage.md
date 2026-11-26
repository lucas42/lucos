# Incident: Estate-wide outage after a dockerd restart, kept down by lucos_firewall's full-table rollback

| Field | Value |
|---|---|
| **Date** | 2026-11-25 |
| **Duration** | ~35h47m (2026-11-25 06:15 UTC to 2026-11-26 ~18:03 UTC) |
| **Severity** | Complete outage |
| **Services affected** | Every router-fronted service on avalon (all `*.l42.eu` vhosts served there, incl. configy, monitoring, contacts, creds, media, aithne). xwing's public ports intermittently (~40s of every ~2 min). Anything on xwing/salvare depending on avalon-hosted services |
| **Detected by** | User report. lucas42 couldn't reach lucOS services from abroad. No alert was delivered (see lucas42/lucos#334) |

Source issue: lucas42/lucos_firewall#31

---

## Summary

A routine unattended `docker-ce` upgrade restarted dockerd on avalon at 06:15:44 UTC on 2026-11-25. Twelve seconds later, lucos_firewall's poll found configy unreachable mid-restart, snapshotted the host's iptables, and entered an apply-fallback, confirm, revert loop that it could never leave: configy sits behind the router, and the router depends on the Docker NAT. The revert restores *every* table from the snapshot, including Docker's `nat` table. avalon was left with **zero DNAT rules for its 38 published ports**, re-imposed every ~2 minutes for the next 35 hours, so lucos_router couldn't reach any backend. Monitoring detected the outage throughout but couldn't deliver a single alert, because its mail relay and loganne both sit behind the same NAT. Service was restored by freezing the firewall in `DRY_RUN` and restarting dockerd, which re-created all 38 DNAT rules. The override was then reverted with a normal CircleCI redeploy.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-11-25 06:13:20 | xwing: unattended upgrade `docker-ce` 29.10.0 → 29.10.1; dockerd restarts 06:13:52 |
| 06:15:32 | avalon: same upgrade; dockerd restarts 06:15:44 (`live-restore: true`, containers stay up) |
| 06:15:47 | lucos_monitoring logs its first `Not sending alert … only 1 recurring failures` lines for many systems |
| 06:15:55 | avalon lucos_firewall: configy fetch `connect: connection refused`, so it applies the fallback ruleset. Its snapshot is taken here, **12s after the dockerd restart** |
| 06:16:34 | xwing lucos_firewall: first configy failure (configy is on avalon), enters the same loop. Its snapshot is **2m42s after** xwing's dockerd restart, and its NAT survives |
| 06:16:36 | avalon: confirmation fails and it reverts to the snapshot. From here the loop repeats every ~2 min on both hosts |
| 06:20:52 | lucos_monitoring's first `Send notifications`. 267 follow over the outage; 28 email sends fail with `network_failure,"178.32.218.44",{error,timeout}` |
| 2026-11-26, before ~17:37 (exact time not recorded) | lucas42 reports from abroad that lucOS is unreachable; team-lead does first checks |
| ~17:37 | SRE engaged: avalon pages time out from xwing after TLS completes; router logs `upstream timed out … while connecting to upstream` for every backend |
| ~17:42 | Host-local probes of backend ports return 200, so the failure is isolated to router→host. Firewall loop found in lucos_firewall logs |
| ~17:47 | avalon `nat` table found to contain **0 DNAT rules for 38 published ports**. xwing's NAT is intact (6/6) |
| 18:01:00 | lucos_firewall on avalon frozen with `DRY_RUN=true` via a one-off `docker compose up -d` (lucas42-approved), gated to fire just after a revert. iptables hashes confirmed unchanged |
| 18:02:21 | lucas42 runs `systemctl restart docker` on avalon |
| ~18:03 | **Service restored.** 38/38 DNAT rules, router upstream errors stop, avalon pages return 200 from xwing over IPv4 and IPv6 |
| 18:04:30 | xwing lucos_firewall: "Rules confirmed and active"; loop ends. salvare confirmed enforcing normally |
| 18:13:42 | lucos_firewall v1.0.21 deployed via CircleCI (pipeline 70), removing the `DRY_RUN` override. avalon "Rules confirmed and active" at 18:14:12; DNAT still 38 |
| TBD | Scheduled-job checks left over from the outage clear on their next runs (TBD pending re-check) |

---

## Analysis

### Stage 1: a routine dockerd restart

The trigger was mundane. `unattended-upgrades` installed a docker-ce point release on avalon and xwing within two minutes of each other. xwing's dpkg log shows the previous point release arriving the same way on 2026-11-17. With `live-restore: true` the containers kept running, and on its own the restart should have been a ~10-second blip. Whether docker-ce should be in the automatic-upgrade set at all has been raised with lucos-system-administrator.

### Stage 2: the firewall took its snapshot mid-restart

lucos_firewall's `applyWithRollback()` takes a snapshot with `iptables-save` / `ip6tables-save` (all tables) before every apply. On avalon its next poll fell 11s after dockerd came back. configy answered `connection refused`, because the router was briefly unreachable during the restart, so the firewall went to its base-only fallback and took the snapshot.

What isn't proven is *how* the DNAT rules were lost. There are two candidate orderings:
- the snapshot was taken before the restarted dockerd had re-programmed port DNAT; or
- dockerd re-programmed them and a subsequent full-table revert overwrote them.

Either way the end state is the same. The cross-host comparison favours the first: xwing's snapshot came 2m42s after its restart and its NAT survived, while avalon's came 12s after and its NAT was lost. This hasn't been reproduced.

### Stage 3: a loop with no exit

Three properties made the transient permanent:

1. **The revert is full-table.** `runRestore(..., noflush=false)` replaces every table from the snapshot, including Docker's `nat` table and Docker's own filter chains. The firewall *owns* only INPUT and DOCKER-USER. Every ~2 minutes it re-imposed a broken Docker state.
2. **It fails closed on its own control plane.** configy being unreachable makes it change rules (apply the fallback), even though the last applied ruleset had been confirmed. Enforce *mode* is held last-known-good across configy blips, but the *ports* are not.
3. **The confirmation check sits behind the thing it's checking.** On avalon, configy is reached through lucos_router, which needs the Docker NAT, which the revert keeps breaking. So the loop can never confirm.

Without DNAT, the router's connections to `172.17.0.1:<port>` landed on the host's INPUT chain (policy DROP), not FORWARD, and timed out. This happened even in the reverted windows, when the full 7-port ruleset and :443 were live. The backends themselves were healthy the whole time: host-local probes returned 200.

xwing ran the same loop but its snapshot was healthy. So its reverts restored working NAT, and the impact there was limited to the fallback windows closing its public ports.

### Stage 4: detection worked, delivery didn't

lucos_monitoring, on avalon, stayed up and detected the outage within seconds. It logged 267 `Send notifications` events. None were delivered:
- the SMTP relay is on avalon (28 `retries_exceeded … network_failure … timeout`);
- loganne is behind the same router.

Monitoring's own outbound connections also appear to have failed: ~2,100 CircleCI API timeouts per repo during the outage, and 0 afterwards. This is the risk lucas42 accepted in lucas42/lucos#295 (closed not-planned 2026-10-18), but triggered by something far more ordinary than "avalon goes down hard": avalon was up, SSH worked, and containers kept running. Re-raised for decision as lucas42/lucos#334.

The outage lasted 35h47m because nobody knew it was happening.

---

## What Was Tried That Didn't Work

- **Probing from the agent sandbox.** It was on unreliable train WiFi, and its timeouts and DNS stalls were indistinguishable from the outage. xwing was used as the vantage point instead, at lucas42's suggestion.
- **"The router network was recreated, so the firewall's bridge allow-rule is stale."** This was my first hypothesis, because the router (up 3 days) looked younger than the firewall (up 4 weeks). It was wrong: the network dated from 2026-09-16, and the firewall has no per-bridge INPUT rule anyway.
- **Option (a) alone, i.e. setting `DRY_RUN`.** It was initially proposed as the fix. Reading the code showed dry-run leaves iptables exactly as it is, so it would have frozen the broken state, not repaired it. It became step 1 of 2, with the dockerd restart as step 2.
- **`sudo iptables` / `nft` on avalon.** The agent has no sudo, so the rules were read read-only via `docker exec lucos_firewall iptables-save`.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Scope the firewall's rollback to owned chains; don't change confirmed rules because configy is unreachable; reconsider the confirmation target | lucas42/lucos_firewall#31 | Open |
| Decide whether to re-accept the alerting-on-avalon risk given this incident | lucas42/lucos#334 | Open (Awaiting Decision) |
| Decide whether docker-ce/containerd should be in unattended-upgrades on production hosts | TBD (raised with lucos-system-administrator) | TBD |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

The firewall was in a mixed state during the incident, and during the ~12 minutes of `DRY_RUN` (18:01–18:13) avalon's ruleset was static rather than reconciled against configy. That ruleset was the normal configy-derived one throughout, so no ports were opened beyond what is normally public.
