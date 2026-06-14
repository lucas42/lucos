# Incident: photos_photos volume left without a fresh backup for ~6 days during incremental-rsync rollout

| Field | Value |
|---|---|
| **Date** | 2026-06-08 |
| **Duration** | ~6 days data-at-risk (2026-06-08 → 2026-06-14 08:45 UTC, first successful incremental backup) |
| **Severity** | Partial degradation / Data risk (single volume; staleness, not data loss) |
| **Services affected** | lucos_backups — `lucos_photos_photos` volume only; all other volumes backed up normally throughout |
| **Detected by** | SRE — originating issue lucas42/lucos_backups#309 (proactive), resolution driven through the incremental-rsync rollout |

Originating issues: lucas42/lucos_backups#309 (the backup gap) and the incremental-backup work (lucas42/lucos_backups#324, implementing ADR-0002).

---

## Summary

The 6.6 GB `lucos_photos_photos` volume outgrew the hardcoded 600-second `scp`-to-aurora timeout in the full-snapshot backup path, so its daily backup began failing on/around 2026-06-08 (lucas42/lucos_backups#309). A prior snapshot existed throughout, so this was backup *staleness* (a growing RPO), not data loss — but the volume received no fresh backup for ~6 days. The fix was to switch this volume to an incremental `rsync --link-dest` strategy (ADR-0002, lucas42/lucos_backups#324). That new path then failed its first two live exercises in series — first on a ProxyJump host-key bug (lucas42/lucos_backups#327, fixed in v1.1.14), then on an atomic-publish shell-quoting bug that stranded the transferred data (lucas42/lucos_backups#330, fixed in v1.1.15). The incremental path completed end-to-end after v1.1.15 deployed at 2026-06-14 08:45 UTC; the `create-backups` monitoring check — which fails if any volume's backup errors — has been green since.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~2026-06-08 | `lucos_photos_photos` daily full-snapshot backup failing — 6.6 GB exceeds the hardcoded 600 s `scp`-to-aurora timeout (avalon → ProxyJump xwing → aurora). Prior snapshot exists; no fresh backup from here on. |
| 2026-06-08 16:07 | lucas42/lucos_backups#309 filed (P2): `create-backups` red for photos_photos (timeout) and, separately, media_manager_stateFile (tar short read). Neither caused by the firewall enforce. |
| 2026-06-10 16:20 | lucas42/lucos_backups#324 opened — incremental `rsync --link-dest` strategy for the photos volume (ADR-0002), per-volume opt-in; full-snapshot stays the default for every other volume. |
| 2026-06-12 23:41 | lucas42/lucos_backups#324 merged. |
| 2026-06-12 23:45 | lucos_backups **v1.1.13** deployed to avalon (incremental path live for photos_photos, with configy#230's `backup_strategy` field). |
| 2026-06-12 23:53 | lucas42/lucos_backups#327 filed — first live seed of the incremental path fails: ProxyJump hop fails host-key verification (empty `known_hosts` in the ephemeral rsync container). |
| 2026-06-13 23:43 | lucas42/lucos_backups#329 (fix) merged. |
| 2026-06-13 23:45 | lucos_backups **v1.1.14** deployed — rsync now transfers successfully, which exposes the next bug. |
| 2026-06-14 08:29 | lucas42/lucos_backups#330 filed — distinct second bug: the atomic-publish `rm && mv` runs `mv` on the wrong host; transferred data stranded in `<date>.partial/`, never published. |
| 2026-06-14 08:42 | lucas42/lucos_backups#331 (fix) merged. |
| 2026-06-14 08:45 | lucos_backups **v1.1.15** deployed — incremental path completes end-to-end; photos_photos backup published. |
| 2026-06-14 (this ops check) | `create-backups` monitoring check green; photos_photos backing up via the incremental path. |

---

## Analysis

### Originating cause: a 6.6 GB volume against a 600 s transfer timeout

The full-snapshot path `tar`s a volume and `scp`s the archive to aurora over a ProxyJump chain (avalon → xwing → aurora) under a hardcoded 600-second timeout. `lucos_photos_photos` grew past what 600 s allows, so every daily run timed out mid-transfer and produced no snapshot (lucas42/lucos_backups#309). This is the failure mode that *worsens with time* — every photo added moves the volume further past the cliff — which is why the durable fix was an incremental strategy rather than simply bumping the timeout.

### The incremental path's two serial latent bugs

The `rsync --link-dest` path (lucas42/lucos_backups#324) ran the rsync inside an ephemeral `docker run` container as root, transferring to aurora across the same ProxyJump gateway. It carried two latent bugs that could only surface on a real run, and surfaced *one at a time*:

1. **ProxyJump host-key verification (lucas42/lucos_backups#327 → #329).** The rsync container has an empty `known_hosts`, and a command-line `ssh -o StrictHostKeyChecking=no` does **not** propagate to the ProxyJump hop — so the jump to the gateway failed host-key verification (`Host key verification failed` / rsync exit 255). The transfer never started. Fixed by mounting the source host user's `known_hosts` read-only into the rsync container (v1.1.14). `StrictHostKeyChecking=no` is **retained** for parity with the host-side path — so a *known* key is now verified, but a *changed* key is still accepted silently; tightening that to `accept-new` is tracked as a non-blocking hardening (lucas42/lucos_backups#332).

2. **Atomic-publish command runs on the wrong host (lucas42/lucos_backups#330 → #331).** The publish step `rm -rf <final> && mv <partial> <final>` was passed **unquoted** through `runOnRemote`, so the source host's local shell interpreted the `&&`: `rm -rf` ran on the *target* (via ssh) but `mv` ran *locally* on the source, where the `.partial` directory doesn't exist. rsync succeeded and the data landed in `<date>.partial/` on the target, but was never published to the final `<date>/` snapshot — so the run was reported as failed and produced no usable backup. Fixed by `shlex.quote`-ing the command in `runOnRemote` so the whole compound is evaluated on the target (v1.1.15). The fix lives in the shared `runOnRemote` helper (`src/classes/host.py`), not at the photos call site — so it closes this class for **every** caller, current and future, and no separate audit of other `runOnRemote` callers is required. (Thanks to lucos-developer for raising the broader-caller question during review.)

### Why the bugs surfaced serially, and why CI didn't catch them

Bug 1 blocked rsync entirely, so bug 2 (in the post-transfer publish step) was unreachable until bug 1 was fixed. The deeper reason both shipped at all is that this path — rsync from an ephemeral container, over a ProxyJump gateway, to aurora — **cannot be exercised in CI** without the real gateway and aurora reachable, so it was first exercised only by the off-cron seed run against production. A code path that is only ever run live will hide bugs in series: each fix unblocks the next failure.

---

## What Was Tried That Didn't Work

- **Assuming `StrictHostKeyChecking=no` alone would carry the container's SSH.** The cross-host path already sets `StrictHostKeyChecking=no` (host-side parity), and the natural assumption was that this would let the empty-`known_hosts` rsync container connect without host-key friction. It doesn't: `StrictHostKeyChecking=no` is **not** propagated to the ProxyJump hop, so the jump connection still failed verification (#327). The fix was not to *remove* `=no` but to *add* a mounted `known_hosts` so the jump host's key is known; `=no` is retained as the first-use (TOFU) fallback — which is exactly why a follow-up hardening to `accept-new` is now tracked (see Follow-up Actions).
- Diagnosis was otherwise first-attempt for each bug — the error text named the failure mode directly (`Host key verification failed`; transferred data sitting in `<date>.partial/`).

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Incremental `rsync --link-dest` strategy for the photos volume (ADR-0002) | lucas42/lucos_backups#324 | Done (v1.1.13) |
| Fix ProxyJump host-key verification (mount host `known_hosts` into the rsync container) | lucas42/lucos_backups#329 | Done (v1.1.14) |
| Shell-quote `runOnRemote` so the incremental publish `rm`/`mv` runs on the target | lucas42/lucos_backups#331 | Done (v1.1.15) |
| Harden cross-host backup SSH to `StrictHostKeyChecking=accept-new` (reject *changed* host keys, not just verify known ones) — raised by lucos-security during review | lucas42/lucos_backups#332 | Open (P3, non-blocking hardening) |
| Build a CI/staging harness exercising the incremental rsync-over-ProxyJump path before deploy | — | **Not filed — risk accepted.** The path needs the real gateway + aurora reachable and would be non-deterministic in CI (high build/maintain cost); the failure mode is an internal backup delayed by ≤1 day, and the existing `create-backups` monitoring check already catches both a hard failure and the data-stranding case (the run reports failed → check goes red) within a day. The effort doesn't justify the marginal detection gain. Captured here rather than as a low-value ticket. |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

(The incremental path uses SSH/ProxyJump to aurora with a key held by the source host user; no key material or credential is exposed or reproduced here.)
