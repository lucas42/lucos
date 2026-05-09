# Incident: lucos_backups SSH key rotation broke authentication on every host

| Field | Value |
|---|---|
| **Date** | 2026-05-09 |
| **Duration** | ~1 hour 14 minutes (00:42:54 UTC to 01:56:37 UTC) |
| **Severity** | Complete outage of backup tracking (data risk if not recovered before the next daily backup window) |
| **Services affected** | lucos_backups |
| **Detected by** | monitoring alert (`host-tracking-failures`, `volume-host` on `backups.l42.eu`) |

---

## Summary

A routine SSH key rotation for the `lucos-backups` user left every backup-target host's `authorized_keys` file holding the old public key. The `rotate-ssh-key.sh` script writes the new keypair to lucos_creds but does not push the new public key to any host. Three of the four hosts (avalon, salvare, xwing) were unstuck by re-running `init-host.sh`. The fourth (aurora.local, a kernel-3.4.6 armv5tel QNAP box with both busybox userland and a non-`/home` user-home layout — `/share/homes/$user`) failed at multiple points: the original `init-host.sh` never ran end-to-end on it (it had a manually-bootstrapped `.ssh/` directory at the QNAP-style path dating back to its original setup), and the new `update-authorized-keys.sh` script introduced in the recovery (PR lucas42/lucos_backups#267) initially had the same `/home/${USERNAME}/` path-hardcoding bug. After a path-expansion fix to the new script, lucas42 ran it against aurora and authentication recovered. No backup data was lost — the daily backup window had not yet fired — but the tracking daemon was unable to enumerate any host's volumes for the duration, and the next backup run would have failed on every host had the auth not been restored in time.

---

## Timeline

All times UTC.

| Time | Event |
|---|---|
| 00:33 | `SSH_PRIVATE_KEY` updated in `lucos_backups (development)` (probe / format check, related to lucas42/lucos_backups#262 cleanup) |
| 00:39:23–24 | `SSH_PRIVATE_KEY` and `SSH_PUBLIC_KEY` updated in **both** dev and prod environments — the four-event signature of `rotate-ssh-key.sh` running |
| 00:39:56 | lucas42/lucos_backups#265 merged (removed the `~`/`=` substitution workaround) |
| 00:42:00 | v1.0.62 deployed to avalon |
| 00:42:54 | Monitoring alert: `2 failing check(s) on lucos backups (backups.l42.eu): host-tracking-failures, volume-host` — daemon failed to authenticate to any of `aurora.local`, `avalon.s.l42.eu`, `salvare.s.l42.eu`, `xwing.s.l42.eu`. **Incident starts** |
| 00:55:31 | Monitoring alert reduces from 2 failing to 1 failing — `volume-host` clears (cached volume-by-host data was still within the daemon's 2-hour freshness window, satisfying the check from cache); `host-tracking-failures` continues firing on all four hosts |
| ~01:00 | lucas42 reports the breakage; SRE engaged |
| ~01:00–01:15 | SRE confirms root cause: new private key parses fine and is loaded into the daemon's ssh-agent (container log: `Identity added: (stdin) (lucos_backups)`), but every host returns "Authentication failed." — the new public key was never pushed to any host's `authorized_keys` |
| ~01:15 | lucas42 begins re-running `init-host.sh` per host. avalon, salvare, xwing succeed and re-authenticate within minutes; aurora.local fails |
| ~01:18 | `init-host.sh admin@aurora.local` fails: `sudo: usermod: command not found`. SRE diagnostic identifies aurora as busybox-based (kernel 3.4.6 armv5tel, no shadow-utils, only busybox `adduser`/`addgroup` in `/bin`) |
| ~01:18 | (No `monitoringRecovery` event yet — `host-tracking-failures` is a single check covering all four hosts, so it stays firing until aurora is also fixed, even though three of four are now authenticating) |
| ~01:21 | SRE files lucas42/lucos_backups#266 (split key-distribution out of `init-host.sh`); lucas42 confirms he'll wait for the new script rather than running an ad-hoc unstick |
| 01:24:54 | Developer commits initial version of `update-authorized-keys.sh` on PR lucas42/lucos_backups#267 (still has the inherited `/home/${USERNAME}/` path hardcoding) |
| 01:35–01:37 | Developer iterates on argument handling and an `ssh user@user@host` quoting bug in the auth-test step |
| ~01:40–01:45 | lucas42 runs `update-authorized-keys.sh` against aurora and hits `tee: /home/lucos-backups/.ssh/authorized_keys: No such file or directory` |
| ~01:45 | SRE diagnostic identifies aurora's user home as `/share/homes/lucos-backups` (QNAP path layout, not `/home/`); the script inherited the same path-hardcoding bug from `init-host.sh` |
| ~01:47 | SRE proposes the one-line fix (replace `/home/${USERNAME}/` with `~${USERNAME}/` shell expansion plus a `.ssh/` pre-check) |
| 01:47:45 | Developer commits the path-expansion fix |
| ~01:55–01:56 | lucas42 runs the path-fixed `update-authorized-keys.sh` against aurora; daemon's next retry succeeds |
| 01:56:37 | All checks healthy on `lucos backups (backups.l42.eu)` (`monitoringRecovery` event). **Incident resolved** |
| 01:57:02 | PR lucas42/lucos_backups#267 merged (auto-merged on the existing approval after the path-fix commit) |

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

### Stage 3 — aurora is QNAP, with both busybox userland and a non-`/home` user-home layout

Aurora is a kernel-3.4.6 armv5tel QNAP NAS. It uses busybox userland (`/bin/adduser`, `/bin/addgroup`, no `usermod` anywhere on the system) **and** the QNAP-style user-home layout (`/share/homes/$user`, not `/home/$user`). Both of these matter, because `init-host.sh` makes two unportable assumptions: it calls shadow-utils binaries (`useradd`, `usermod`) without any portability fallback, and it hardcodes `/home/${USERNAME}/` as the path for the `.ssh/` directory and `authorized_keys` file.

The corollary is that **the original `init-host.sh` never ran end-to-end on aurora**. Aurora's `lucos-backups` user, its `.ssh/` directory, and its working `authorized_keys` file were all set up out-of-band manually at some point years ago, at the QNAP path. The first-init script would have failed on `usermod` first (which is what we saw today during the recovery attempt) and then again on the `mkdir`/`tee` lines for the wrong-path `.ssh/` directory had we got that far. The `usermod`-not-found symptom we tracked today was the *first* line of the script that failed, but it was not the *only* line that would have failed; it just happened to be the earliest one in execution order.

This had two amplifying effects on the recovery:

1. **It removed the obvious recovery path for aurora**, since "re-run `init-host.sh`" — the path that worked for the other three hosts — couldn't run cleanly. We needed a different mechanism, which became the new `update-authorized-keys.sh` script in PR lucas42/lucos_backups#267.
2. **The path-hardcoding bug was inherited by the new script**, because PR #267 was based on the existing `init-host.sh` code shape. lucas42's first attempt at running `update-authorized-keys.sh` against aurora hit `tee: /home/lucos-backups/.ssh/authorized_keys: No such file or directory` for the same reason. A second SRE diagnostic identified the QNAP home layout, the developer amended the script to use `~${USERNAME}/` shell expansion (which expands from `/etc/passwd` and works on every distribution), and the next run succeeded.

The path-hardcoding bug applies equally to `init-host.sh` itself in three places (`mkdir`, `touch`, `tee`); that hardening work is now tracked in lucas42/lucos_backups#268. The busybox `usermod` compatibility is the third sub-concern of that ticket — lower priority because aurora's first-init is years past and there are no other busybox hosts currently in the estate, but worth fixing while everyone is touching the script.

---

## What Was Tried That Didn't Work

- Initial hypothesis (before reading container logs) was that the new `SSH_PRIVATE_KEY` value might still have a format issue from the `~`/`=` substitution being removed in lucas42/lucos_backups#265. The container log line `Identity added: (stdin) (lucos_backups)` ruled that out immediately — the new private key parses fine and is loaded into ssh-agent without complaint. The substitution removal was working correctly; the problem was on the host side, not in the credential.
- For aurora, an early hypothesis was that `sudo` was stripping `/usr/sbin` from PATH (Debian/Ubuntu's default behaviour). The diagnostic ruled that out: aurora's sudo PATH includes `/usr/sbin`, and `usermod` simply doesn't exist anywhere on the filesystem. This is a "package not present" problem (busybox vs shadow-utils), not a PATH problem.
- An initial guess at aurora's distribution was Synology DSM (which has a similar non-`/home` user-home layout at `/var/services/homes/$user`). The diagnostic showed home at `/share/homes/lucos-backups` with a `.Qsync` directory in it, identifying the box as QNAP rather than Synology. Same shape of fix (drop the `/home/` assumption), but the genus was wrong on the first guess.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Split key distribution out of `init-host.sh` into a new `update-authorized-keys.sh`; have `rotate-ssh-key.sh` call it automatically after writing to lucos_creds | lucas42/lucos_backups#266 | Open |
| Implement `update-authorized-keys.sh` (the new script and the `rotate-ssh-key.sh` integration) | lucas42/lucos_backups#267 | Merged 2026-05-09 01:57 UTC |
| Harden `init-host.sh`: fix the same `/home/${USERNAME}/` path hardcoding (three places), delegate the key-write step to `update-authorized-keys.sh` to avoid duplication, and address the busybox `usermod` compatibility for fresh non-shadow-utils hosts | lucas42/lucos_backups#268 | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.

The incident involved an SSH key rotation but no key material is reproduced in this report. The new keypair was generated cleanly by `rotate-ssh-key.sh` (ed25519, freshly random) and stored in lucos_creds in the standard way; no compromise of the prior key is suspected — the rotation was hygiene driven by lucas42/lucos_backups#262 / lucas42/lucos_backups#265.
