# Incident: Estate-wide outage after a dockerd restart, kept down by lucos_firewall's full-table rollback

| Field | Value |
|---|---|
| **Date** | 2026-11-25 |
| **Duration** | ~35h47m (2026-11-25 06:15 UTC to 2026-11-26 ~18:03 UTC) |
| **Severity** | Complete outage |
| **Services affected** | Every router-fronted service on avalon (all `*.l42.eu` vhosts served there, incl. configy, monitoring, contacts, creds, media, aithne). xwing's public ports intermittently (~40s of every ~2 min). Anything on xwing/salvare depending on avalon-hosted services |
| **Detected by** | User report (lucas42). Monitoring detected it within seconds but couldn't deliver an alert (see lucas42/lucos#334) |

Source issue: lucas42/lucos_firewall#31

---

## Summary

A routine unattended `docker-ce` upgrade restarted dockerd on avalon at 06:15:44 UTC on 2026-11-25. Eleven seconds later lucos_firewall's poll found configy unreachable mid-restart. It then entered a loop it could never leave: apply fallback rules, fail to confirm, revert. Each revert restores a snapshot of *every* iptables table, including Docker's `nat` table. avalon was left with **zero DNAT rules for its 38 published ports**, and the loop re-imposed that state every ~2 minutes for the next 35 hours, so lucos_router couldn't reach any backend.

For a user, every avalon-hosted page connected and completed TLS, then hung until the client gave up, or occasionally returned a 504. That looks exactly like a bad network connection, which is why the outage couldn't easily be told apart from flaky WiFi.

Monitoring detected the outage within seconds but couldn't deliver a single alert, because its mail relay and loganne both sit behind the same NAT. Service was restored by freezing the firewall in `DRY_RUN` and restarting dockerd, which re-created all 38 DNAT rules. The override was then removed with a normal CircleCI redeploy.

The fix the firewall needed was already a recorded decision. ADR-0007 Amendment 1 requires it to hold its last-known-good ports through a configy outage, but only the enforce-mode half of that contract was ever built.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-11-04, 11-05, 11-17, 11-18 | Earlier unattended docker-ce upgrades restart dockerd on avalon (11-04, 11-18) and xwing (11-05, 11-17) without incident. Dates from lucos-system-administrator. Firewall logs for those dates not checked |
| 2026-11-25 06:13:20 | xwing: unattended upgrade `docker-ce` 29.10.0 → 29.10.1; dockerd restarts 06:13:52 |
| 06:15:32 | avalon: same upgrade, 2m12s after xwing's; dockerd restarts 06:15:44 (`live-restore: true`, containers stay up) |
| 06:15:47 | lucos_monitoring logs its first `Not sending alert … only 1 recurring failures` lines for many systems |
| 06:15:55 | avalon lucos_firewall poll: configy fetch `connect: connection refused` (**11s after the dockerd restart**) |
| 06:15:56 | avalon lucos_firewall snapshots all tables (`applyWithRollback`) and applies the base-only fallback ruleset (**12s after the restart**) |
| 06:16:34 | xwing lucos_firewall: first configy failure (configy is on avalon); enters the same loop. Its snapshot is **2m42s after** xwing's dockerd restart, and its NAT survives |
| 06:16:36 | avalon: confirmation fails (configy unreachable), so it reverts to the snapshot. From here the loop repeats every ~2 min |
| 06:20:52 | lucos_monitoring's first `Send notifications`. 267 follow over the outage; 28 email sends fail with `network_failure,"178.32.218.44",{error,timeout}` |
| 2026-11-26, before ~17:37 (exact time not recorded) | lucas42 reports that lucOS is unreachable; team-lead does first checks |
| ~17:37 | SRE engaged: avalon pages time out from xwing after TLS completes; router logs `upstream timed out … while connecting to upstream` for every backend |
| ~17:42 | Host-local probes of backend ports return 200, so the failure is isolated to router→host. Firewall loop found in lucos_firewall logs |
| ~17:47 | avalon `nat` table found to contain **0 DNAT rules for 38 published ports**. xwing's NAT is intact (6/6) |
| 18:01:00 | lucos_firewall on avalon frozen with `DRY_RUN=true` via a one-off `docker compose up -d` (lucas42-approved), gated to fire just after a revert. iptables hashes confirmed unchanged against a baseline |
| 18:02:21 | lucas42 runs `systemctl restart docker` on avalon |
| ~18:03 | **Service restored.** 38/38 DNAT rules, router upstream errors stop, avalon pages return 200 from xwing over IPv4 and IPv6 |
| 18:04:30 | xwing lucos_firewall: "Rules confirmed and active"; loop ends. salvare confirmed enforcing normally |
| 18:13:42 | lucos_firewall v1.0.21 deployed via CircleCI (pipeline 70), removing the `DRY_RUN` override. avalon "Rules confirmed and active" at 18:14:12; DNAT still 38 |
| 18:53 | lucos_time's contacts cache refreshes; monitoring at 54/56 healthy. Remaining reds are scheduled jobs that last ran during the outage |
| 18:58 | lucos_time's eolas cache refreshes on its hourly cycle |
| 19:04–19:24 | Remaining reds actively cleared, at lucas42's request: lucos_repos audit sweep triggered; **21** Dependabot PRs found stuck (monitoring's "3" predated the outage). 19 had `Docker Login (mirror)` timeouts against docker.l42.eu during the outage; one had a transient Go-proxy failure from 11-24; one (lucas42/.github#78) needed code-reviewer approval. CircleCI workflows re-run from failed; all 21 merged by 19:24 |
| 19:18:26 | lucas42/lucos_firewall#32 (fix for lucas42/lucos_firewall#31) merges |
| 19:24–19:37 | lucos_firewall v1.0.22 deployed and confirmed on salvare (19:24:18), xwing (19:25:39) and avalon (19:37:51). On xwing and avalon the nat table, DNAT counts (6, 38) and owned chains are byte-identical to a pre-deploy baseline |
| 19:34 | **Monitoring 56/56 healthy.** Incident resolved |
| TBD | Namespace-isolated test of v1.0.22's revert path on xwing (TBD pending result) |

---

## Analysis

### Stage 1: a routine dockerd restart

The trigger was mundane. `unattended-upgrades` installed a docker-ce point release on xwing and then on avalon, 2m12s apart. Docker auto-upgrade is deliberate: lucas42/lucos_agent_coding_sandbox#100 extended unattended-upgrades to Docker packages with "no carve-outs", on the assumption that `live-restore: true` makes daemon restarts harmless. On its own, this restart was the ~10-second blip that assumption predicts. According to lucos-system-administrator, four earlier docker-ce upgrades this month were exactly that. Whether docker-ce/containerd should keep auto-upgrading on production hosts was put to lucas42 (lucas42/lucos_agent_coding_sandbox#122). He chose to keep auto-upgrading: with the firewall fixed, a dockerd restart is back to being the harmless blip the original decision assumed.

### Stage 2: the firewall took its snapshot mid-restart

lucos_firewall's `applyWithRollback()` takes a snapshot with `iptables-save` / `ip6tables-save` (all tables) before every apply. On avalon its next poll fell 11s after dockerd came back. configy answered `connection refused`, because the router was briefly unreachable during the restart. So the firewall went to its base-only fallback and took the snapshot at 06:15:56, 12s after the restart.

What isn't proven is *how* the DNAT rules were lost. There are two candidate orderings:
- the snapshot was taken before the restarted dockerd had re-programmed port DNAT; or
- dockerd re-programmed them and a subsequent full-table revert overwrote them.

Either way the end state is the same. The cross-host comparison favours the first: xwing's snapshot came 2m42s after its restart and its NAT survived, while avalon's came 12s after and its NAT was lost. This hasn't been reproduced. A controlled reproduction would settle it: restart dockerd on a test host and snapshot at +5s versus +3min. The fixes below don't depend on which ordering is true, so I haven't filed that reproduction.

### Stage 3: a loop with no exit

Three properties made the transient permanent:

1. **It changes its rules when it can't reach configy, which ADR-0007 already says it shouldn't.** ADR-0007 Amendment 1, § "Fail-safe contract", says: "On a failed poll, the firewall keeps applying the last successfully-fetched `(mode, ports)` pair." lucas42/lucos_firewall#9 implemented only the *mode* half (`lastKnownEnforce`). The ports path still sets `ports = nil` on a failed fetch and applies the base-only fallback. Following the code path (not reproduced): had last-known-good ports been built, a failed configy fetch would have produced an unchanged ruleset hash, logged "Ruleset unchanged — skipping apply", and never called `applyWithRollback`. That means no snapshot and no revert. **The decision existed; it just wasn't built.**
2. **Apply is scoped; revert is not.** The apply path flushes and restores only the two chains the firewall owns, INPUT and DOCKER-USER, via `--noflush`. Every revert path calls `runRestore(..., noflush=false)`, which replaces whole tables, including Docker's `nat` table and Docker's own filter chains. That was deliberate (the code comment says it is "so Docker's chains are also restored"), and it trusts the snapshot to be good. Here the snapshot wasn't good, so every ~2 minutes the firewall re-imposed a broken Docker state.
3. **It acted on a confirmation probe that was already failing before the change.** The probe goes through the router, which arguably makes it a *better* lockout detector, because it exercises the NAT/DOCKER-USER path. But it had failed before the fallback was applied, so its failure after the apply said nothing about the apply. A confirmation check is only evidence about a change if it passed before the change.

Without DNAT, the router's connections to `172.17.0.1:<port>` landed on the host's INPUT chain (policy DROP), not FORWARD, and timed out. This held even in the reverted windows, when the full 7-port ruleset with :443 open was live; observed at 17:45:55, 11s after a revert. The backends themselves were healthy throughout: host-local probes returned 200.

xwing ran the same loop, but its snapshot was healthy. So its reverts restored working NAT, and the impact there was limited to the fallback windows closing its public ports.

### Stage 4: detection worked, delivery didn't

lucos_monitoring, on avalon, stayed up and detected the outage within seconds. It logged 267 `Send notifications` events. None were delivered:
- the SMTP relay is on avalon (28 `retries_exceeded … network_failure … timeout`);
- loganne is behind the same router.

Monitoring's own outbound connections also appear to have failed: ~2,100 CircleCI API timeouts per repo during the outage, and 0 afterwards. This is the risk lucas42 accepted in lucas42/lucos#295 (closed not-planned 2026-10-18), but triggered by something far more ordinary than "avalon goes down hard": avalon was up, SSH worked, and containers kept running. Re-raised for decision as lucas42/lucos#334.

The outage lasted 35h47m because nobody who could act was told. Monitoring knew within seconds but couldn't reach anyone.

---

## What Was Tried That Didn't Work

- **Probing from the agent sandbox.** Its own network connection was unreliable, so its timeouts and DNS stalls were indistinguishable from the outage. xwing was used as the vantage point instead, at lucas42's suggestion.
- **"The router network was recreated, so the firewall's bridge allow-rule is stale."** This was my first hypothesis, because the router (up 3 days) looked younger than the firewall (up 4 weeks). It was wrong: the network dated from 2026-09-16, and the firewall has no per-bridge INPUT rule anyway.
- **Setting `DRY_RUN` alone.** It was initially proposed as the whole fix. Reading the code showed dry-run leaves iptables exactly as it is, so it would have frozen the broken state, not repaired it. It became step 1 of 2, with the dockerd restart as step 2.
- **`sudo iptables` / `nft` on avalon.** The agent has no sudo, so the rules were read read-only via `docker exec lucos_firewall iptables-save`.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Build ADR-0007's last-known-good *ports* (conformance); make confirmation differential (don't act on a probe already failing pre-apply); scope the revert to owned chains as defence in depth | lucas42/lucos_firewall#31, fixed by lucas42/lucos_firewall#32 (v1.0.22) | Done |
| Decide whether to re-accept the alerting-on-avalon risk given this incident | lucas42/lucos#334 | Open (Awaiting Decision) |
| Decide whether docker-ce/containerd.io keep auto-upgrading on production hosts | lucas42/lucos_agent_coding_sandbox#122 | Done (keep auto-upgrading) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

The firewall's ruleset was in a non-standard state twice: during the outage loop, and during the ~13-minute manual `DRY_RUN` override (18:01:00–18:13:42), when it was static rather than reconciled against configy. What was verified about the override window:

- At 18:01:23, the filter (v4 and v6) and nat hashes matched a pre-freeze baseline: the normal configy-derived 7-port ruleset.
- After the 18:02:21 dockerd restart, which rewrites Docker's own chains, a capture at ~18:04–18:05 showed v4 INPUT policy DROP with the :443 accept, the complete v4 DOCKER-USER chain in its normal form (bridge RETURNs, the 7 configy ports, then DROP), and v6 INPUT policy DROP with :443.
- The full v4 INPUT rule list and the v6 DOCKER-USER chain were **not** enumerated in that window.

So the evidence points to no ports being opened beyond what is normally public, but it doesn't cover every chain. From the 18:14:12 redeploy onwards, the firewall's own "Rules confirmed and active" covers the full ruleset.
