# Incident: lucos_backups SSH key rotation broke authentication on every host

| Field | Value |
|---|---|
| **Date** | 2026-05-09 |
| **Duration** | ~XX minutes (00:42 UTC to HH:MM UTC) — TBD pending aurora.local recovery |
| **Severity** | Complete outage of backup tracking (data risk if not recovered before the next daily backup window) |
| **Services affected** | lucos_backups |
| **Detected by** | monitoring alert (`host-tracking-failures`, `volume-host` on `backups.l42.eu`) |

---

## Summary

A routine SSH key rotation for the `lucos-backups` user left every backup-target host's `authorized_keys` file holding the old public key. The `rotate-ssh-key.sh` script writes the new keypair to lucos_creds but does not push the new public key to any host. Three of the four hosts (avalon, salvare, xwing) were unstuck by re-running `init-host.sh`; the fourth (aurora.local, a busybox/embedded ARM box with no `usermod` binary) needed a manual `authorized_keys` rewrite because `init-host.sh` itself fails on that platform. No backup data was lost — the daily backup window had not yet fired — but the tracking daemon was unable to enumerate any host's volumes, and the next backup run would have failed on every host had the auth not been restored in time.

---

## Timeline

All times UTC.

| Time | Event |
|---|---|
| 00:33 | `SSH_PRIVATE_KEY` updated in `lucos_backups (development)` (probe / format check, related to lucas42/lucos_backups#262 cleanup) |
| 00:39:23–24 | `SSH_PRIVATE_KEY` and `SSH_PUBLIC_KEY` updated in **both** dev and prod environments — the four-event signature of `rotate-ssh-key.sh` running |
| 00:39:56 | lucas42/lucos_backups#265 merged (removed the `~`/`=` substitution workaround) |
| 00:42:00 | v1.0.62 deployed to avalon |
| 00:42:54 | Monitoring alert: `2 failing check(s) on lucos backups (backups.l42.eu): host-tracking-failures, volume-host` — daemon failed to authenticate to any of `aurora.local`, `avalon.s.l42.eu`, `salvare.s.l42.eu`, `xwing.s.l42.eu` |
| ~01:00 | lucas42 reports the breakage; SRE engaged |
| ~01:00–01:15 | SRE confirms root cause: new private key parses fine and is loaded into the daemon's ssh-agent (container log: `Identity added: (stdin) (lucos_backups)`), but every host returns "Authentication failed." — the new public key was never pushed to any host's `authorized_keys` |
| ~01:15 | lucas42 begins re-running `init-host.sh` per host |
| HH:MM | `init-host.sh` succeeds on avalon, salvare, xwing — auth restored on those three hosts — TBD pending exact timestamps |
| ~01:18 | `init-host.sh admin@aurora.local` fails: `sudo: usermod: command not found` |
| ~01:18 | SRE diagnostic identifies aurora as a busybox/embedded ARM box (kernel 3.4.6 armv5tel, no shadow-utils, only busybox `adduser`/`addgroup` in `/bin`) |
| HH:MM | lucas42 unblocks aurora by manually writing the new public key into `/home/lucos-backups/.ssh/authorized_keys` (skipping the busybox-incompatible `usermod` lines) — TBD pending verification |
| HH:MM | All checks healthy on `lucos backups (backups.l42.eu)` — TBD pending recovery event |

---

## Analysis

### Stage 1 — `rotate-ssh-key.sh` is incomplete by design

The rotation script writes the new keypair to lucos_creds and stops. It does not push the new `SSH_PUBLIC_KEY` to any host's `authorized_keys` file. The README treats `rotate-ssh-key.sh` and `init-host.sh` as independent manual setup steps, with no documented "after rotating, re-run `init-host.sh` on every host" instruction.

This wasn't a regression introduced by lucas42/lucos_backups#265 — the procedural gap has been latent in `rotate-ssh-key.sh` since the script was first added (commit `06f645b` in lucas42/lucos_backups). It only bit today because today is the first time a rotation has actually been run. The script's behaviour was never wrong on its own terms; it just expected a follow-up step that nobody had been told they needed to take.

### Stage 2 — `init-host.sh` conflates first-time setup with key distribution

Once the gap was identified, the obvious recovery was "re-run `init-host.sh` on every host." But `init-host.sh` does two distinct jobs in one script:

1. **First-time host setup** — `useradd`, `usermod -G docker`, `usermod -a -G lucos-backups <dailyuser>`, `mkdir /srv/backups`. Run once per host, ever.
2. **Key distribution** — write `SSH_PUBLIC_KEY` to `/home/lucos-backups/.ssh/authorized_keys`. Needs running on every rotation.

Rotation only needs the second. But because the two are bundled, re-running the script on a host that's already been set up causes it to re-attempt the first-time-setup operations. On most hosts those are idempotent and harmless (`useradd` is gated by `id -u` and skipped; `usermod -G` and `usermod -a -G` are idempotent on systems where `usermod` exists). On aurora — where `usermod` doesn't exist at all — they fail hard.

### Stage 3 — aurora is busybox/embedded, not shadow-utils

Aurora is a kernel-3.4.6 armv5tel embedded device (the signature is consistent with a Synology NAS or a similar consumer-grade NAS appliance). It uses busybox: `/bin/adduser`, `/bin/addgroup`, no `usermod` anywhere on the system. The `init-host.sh` script uses shadow-utils binaries (`useradd`, `usermod`) without any portability fallback. Aurora's first-init must have been done out-of-band years ago — the script in its current form would have failed at that time too.

This made the recovery harder than it needed to be: even after we knew the right line of `init-host.sh` to run (the `sudo tee /home/lucos-backups/.ssh/authorized_keys` line), we couldn't run the script as a whole, so the unstick had to be done by hand-copying that one line. The other three hosts didn't have this problem because they're standard Linux distributions where `usermod` exists and the script's idempotency holds.

The busybox compatibility is genuinely out of scope for normal operation — aurora's first-init won't need re-running, and there are no other busybox hosts in the estate. But it amplified today's incident by removing the obvious recovery path (re-run `init-host.sh`) for one of the four hosts.

---

## What Was Tried That Didn't Work

- Initial hypothesis (before reading container logs) was that the new `SSH_PRIVATE_KEY` value might still have a format issue from the `~`/`=` substitution being removed in lucas42/lucos_backups#265. The container log line `Identity added: (stdin) (lucos_backups)` ruled that out immediately — the new private key parses fine and is loaded into ssh-agent without complaint. The substitution removal was working correctly; the problem was on the host side, not in the credential.
- For aurora, an early hypothesis was that `sudo` was stripping `/usr/sbin` from PATH (Debian/Ubuntu's default behaviour). The diagnostic ruled that out: aurora's sudo PATH includes `/usr/sbin`, and `usermod` simply doesn't exist anywhere on the filesystem. This is a "package not present" problem (busybox vs shadow-utils), not a PATH problem.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Split key distribution out of `init-host.sh` into a new `update-authorized-keys.sh`; have `rotate-ssh-key.sh` call it automatically after writing to lucos_creds | lucas42/lucos_backups#266 | Open |
| Make `init-host.sh` busybox-compatible (out of scope today; only matters if a new busybox host is added to the estate — aurora's first-init is years past) | None — separate, lower-priority follow-up. To be filed if/when the estate adds a new busybox host. | Not filed |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

The incident involved an SSH key rotation but no key material is reproduced in this report. The new keypair was generated cleanly by `rotate-ssh-key.sh` (ed25519, freshly random) and stored in lucos_creds in the standard way; no compromise of the prior key is suspected — the rotation was hygiene driven by lucas42/lucos_backups#262 / lucas42/lucos_backups#265.
