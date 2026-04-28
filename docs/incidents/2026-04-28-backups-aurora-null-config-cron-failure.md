# Incident: Backup cron silently failed after Aurora NAS rollout — three bugs across 24 hours from one ADR

| Field | Value |
|---|---|
| **Date** | 2026-04-27 → 2026-04-28 |
| **Duration** | ~9 hours of missing backups (2026-04-28 03:25 UTC first scheduled run failed → 2026-04-28 ~12:30 UTC second ad-hoc rerun completes successfully — TBD pending rerun result). Bug-in-production window ~12.5 hours (v1.0.32 deploy at 2026-04-27 23:59 UTC → v1.0.35 deploy at 2026-04-28 12:00 UTC). |
| **Severity** | Data risk — two consecutive backup cron failures across 24 hours; no `lucos_backups` Loganne event since 2026-04-27 03:58 UTC; schedule-tracker `lucos_backups_errors` reached 2 |
| **Services affected** | `lucos_backups` (cron, `/refresh-tracking`, `/_info` checks `host-tracking-failures` and `volume-host`) |
| **Detected by** | SRE ops check (Bug A); monitoring alert firing on the failed ad-hoc rerun (Bugs B + C) |
| **Source issues** | [lucas42/lucos_backups#221](https://github.com/lucas42/lucos_backups/issues/221) (Bug A — null-config crash) and [lucas42/lucos_backups#226](https://github.com/lucas42/lucos_backups/issues/226) (Bugs B + C — `Repository.backup` hardcoded `/srv/`, recursive ProxyJump) |

---

## Summary

The Aurora NAS integration ([`lucas42/lucos_backups#218`](https://github.com/lucas42/lucos_backups/issues/218) → [`lucas42/lucos_backups#219`](https://github.com/lucas42/lucos_backups/pull/219), deployed as v1.0.32 at 2026-04-27 23:59 UTC) shipped three independent bugs that surfaced in sequence over the following 24 hours, each unblocked by the previous fix:

- **Bug A — Null-config crash** (P1). New `Host` configuration fields (`backup_root`, `shell_flavour`) populated only on the new aurora host. For every other host `lucos_configy` returned `null` for these fields, but the new code used `dict.get(key, default)` which only falls back when the key is *absent* — `None` is treated as a real value. Result: every non-aurora host got `backup_root=None`, the `df -P None …` shell command failed, the overnight `create-backups` cron crashed before emitting a Loganne event, and the hourly `/refresh-tracking` was failing every run. Fixed in [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) by switching to `get(key) or default`.
- **Bug B — `Repository.backup` ignores per-host `backup_root`** (P1). The same scope of changes that threaded `backup_root` through `Volume.backupToAll` left a module-level `ROOT_DIR = '/srv/backups/'` constant in `repository.py` untouched. After Bug A was fixed and the cron got far enough to reach the repo loop, every repo backup attempted `mkdir -p /srv/backups/external/github/repository` on aurora, where the `lucos-backups` user has no permission to create `/srv/`. ~80 repos failed identically.
- **Bug C — `_outbound_ssh_args` recursive ProxyJump** (P1). `Host._outbound_ssh_args` added `ProxyJump=<gateway>` to the SSH command whenever the target had an `ssh_gateway` configured, without checking whether the source host running the SSH command was already that gateway. When xwing tried to copy a volume to aurora, xwing was told to ProxyJump *through xwing* to reach aurora — a recursive connection that fails with SSH error 255. All 3 xwing-source volume → aurora copies failed.

Bugs B and C were both fixed in [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) (merged 2026-04-28 11:58 UTC, deployed as v1.0.35 at 12:00 UTC). The second ad-hoc rerun verified all three failure modes resolved — TBD pending rerun result.

All three bugs trace to a single ADR rollout (ADR-0001, aurora NAS integration) with insufficient cross-host test coverage. The dev/prod parity gap (Bug A) and the multi-host parameterisation gap (Bugs B + C) are recurring classes of issue with this codebase that warrant follow-up work tracked separately.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-27 18:29 | [`lucas42/lucos#108`](https://github.com/lucas42/lucos/pull/108) (ADR-0001 for Aurora NAS integration) merged |
| 2026-04-27 23:57 | [`lucas42/lucos_backups#219`](https://github.com/lucas42/lucos_backups/pull/219) "Implement aurora NAS integration" merged (commit `1d55b5e`, introduced all three bugs) |
| 2026-04-27 23:59 | `lucos_backups` v1.0.32 deployed to avalon |
| 2026-04-28 03:25 | Scheduled `create-backups` cron run **failed silently** (Bug A) — crashed in `Host.__init__`, no `backups` Loganne event emitted |
| 2026-04-28 03:25 → 09:49 | Hourly `/refresh-tracking` cron failing every run; `host-tracking-failures` and `volume-host` checks both `ok=false` |
| 2026-04-28 07:14 | `lucos_backups` v1.0.33 deployed (unrelated fix — added regression tests for `_outbound_ssh_args`; did not address Bug A and did not detect Bug C, since the new tests only covered the source-not-gateway path) |
| 2026-04-28 09:49 | SRE ops check opens [`lucas42/lucos_backups#221`](https://github.com/lucas42/lucos_backups/issues/221) (P1) — Bug A root cause identified by reproducing inside the production container |
| 2026-04-28 09:58 | [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) merged with the Bug A fix |
| 2026-04-28 10:00 | `lucos_backups` v1.0.34 deployed; `host-tracking-failures` and `volume-host` recover; `host-count=4`, `volume-count=34` |
| 2026-04-28 ~11:07 | First ad-hoc `create-backups` triggered manually (since the scheduled run had been missed) |
| 2026-04-28 11:22 | First ad-hoc run **fails** — Bug B fires (every repo, ~80 entries) and Bug C fires (3 xwing-source volume → aurora copies). schedule-tracker `lucos_backups_errors=2`. lucos_photos_photos succeeds on the avalon→aurora path (avalon is not the gateway, so Bug C does not apply) |
| 2026-04-28 ~11:30 | Monitoring alert relayed to SRE; second-failure investigation begins |
| 2026-04-28 11:51 | [`lucas42/lucos_backups#226`](https://github.com/lucas42/lucos_backups/issues/226) opened (P1) covering Bugs B and C |
| 2026-04-28 11:52 | [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) opened with the bundled fix and 5 new regression tests |
| 2026-04-28 11:58 | [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) approved (reviewer + lucas42) and merged |
| 2026-04-28 12:00 | `lucos_backups` v1.0.35 deployed |
| 2026-04-28 12:01 | Second ad-hoc `create-backups` triggered |
| 2026-04-28 HH:MM | Second ad-hoc run completes — TBD pending rerun result (will be filled in on completion) |

---

## Analysis

### Bug A: The `dict.get(key, default)` / configy explicit-null mismatch

`Host.__init__` (`src/classes/host.py:29`) read the new optional config:

```python
self.backup_root = host_config.get("backup_root", "/srv/backups/")
```

This idiom relies on the key being **absent** to fall back to the default. But `lucos_configy` does not omit unset optional fields — it serialises them as explicit `null`. When `Host.__init__` asked for `backup_root`, it got `None`, not the fallback string. The same shape applied to `shell_flavour` (which by accident remained benign — `if shell_flavour == "busybox"` correctly fell through to `GnuShell`).

In `GnuShell.disk_space()` (`src/classes/shell.py:55`), the `None` ended up substituted into a shell command:

```python
free_bytes = int(self.connection.run(
    "df -P {backup_root} | tail -1 | awk '{{print $4}}'".format(backup_root=self.backup_root),
    ...
).stdout.strip())
```

Producing `df -P None | tail -1 | awk '{print $4}'` — `df` errored with "No such file or directory" on stderr, stdout was empty, and `int('')` raised `ValueError: invalid literal for int() with base 10: ''`. That exception propagated out of every per-host iteration, leaving `host-count=1` (only aurora, where `backup_root` is set) and `volume-count=0` (nothing tracked on the failing hosts).

### Why this slipped through CI

The most likely contributing factor: [`lucas42/lucos_backups#219`](https://github.com/lucas42/lucos_backups/pull/219) was tested locally against the YAML file at `~/sandboxes/lucos_configy/config/hosts.yaml`, where optional fields are simply omitted from the host record. That shape behaves correctly with `dict.get(key, default)`. The configy HTTP API normalises the response to include every defined field (with `null` for unset ones), and that shape is what production sees. There was no test exercising the configy API shape directly, only the underlying YAML.

This is a recurring class of dev/prod parity issue with configy that is worth eliminating with a fixture / contract test.

### Why detection took ~7 hours

The cron job at 03:25 UTC failed before reaching the point that publishes the `backups` Loganne event, so no monitoring webhook fired. The schedule-tracker check on `lucos_backups` tolerates 2 consecutive errors before flagging — last night's run incremented the count from 0 to 1, which does not fire an alert. The `/_info` endpoint reported `host-tracking-failures: ok=false` and `volume-host: ok=false` continuously from ~03:25 onwards, which monitoring **did** see — but those particular checks were already flapping during the deploy wave (suppressed during the deploy window), so the post-suppression alert was indistinguishable from the noise around the v1.0.32 → v1.0.33 deploy churn until the morning ops check sat down with the actual `/_info` body and dug in.

### Bug B: `Repository.backup` ignored per-host `backup_root`

`lucas42/lucos_backups#218`'s scope explicitly enumerated which call sites needed `backup_root` threading: `getOneOffFiles`, `checkDiskSpace`, `checkBackupFiles`, `getBackups`, and `Volume.backupToAll`. **`Repository.backup` was not in that list.** The implementation in `lucas42/lucos_backups#219` correspondingly left a module-level `ROOT_DIR = '/srv/backups/'` constant in `src/classes/repository.py` and used it inside the per-host loop:

```python
ROOT_DIR = '/srv/backups/'
...
def backup(self):
    directory = "{ROOT_DIR}external/github/repository".format(ROOT_DIR=ROOT_DIR)
    ...
    for host in Host.getAll():
        host.connection.run("mkdir -p {directory}".format(directory=directory), ...)
        host.connection.run("wget ... -O {archivePath}", ...)
```

After Bug A was fixed and the cron got far enough to reach the repository loop, this hardcoded path was attempted on every host including aurora. Aurora's `lucos-backups` user does not have permission to create `/srv/`, so the `mkdir` failed with:

```
mkdir: Cannot create directory `/srv/': Permission denied
```

All ~80 repos failed identically. The fix in [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) drops `ROOT_DIR` and moves the directory/archivePath construction inside the `for host in Host.getAll()` loop, using `host.backup_root`.

### Bug C: `_outbound_ssh_args` recursive ProxyJump (source-is-gateway edge case)

`Host._outbound_ssh_args(target_host)` (`src/classes/host.py:86`) added the `-o ProxyJump=<gateway>` option whenever `target_host.ssh_gateway` was set:

```python
def _outbound_ssh_args(self, target_host):
    args = ['-o', 'StrictHostKeyChecking=no']
    if target_host.ssh_gateway:
        args += ['-o', 'ProxyJump={}'.format(target_host.ssh_gateway_domain)]
    return args
```

`Host.runOnRemote` then uses `self.connection.run(...)` to execute that SSH command **on the source host** (`self`). When `self` is xwing and `target_host` is aurora, the resulting command is `ssh -o ProxyJump=xwing.s.l42.eu aurora.local mkdir -p /share/...` — running on xwing, asking xwing to ProxyJump *through xwing* to reach aurora. SSH errors out with exit code 255 ("Could not connect"). All 3 xwing-source volume → aurora copies (`lucos_media_import_state`, `lucos_router_generatedconfig`, `lucos_router_letsencrypt`) failed this way. avalon-source copies were unaffected because avalon is not the gateway, so the ProxyJump=xwing flag routed correctly.

The fix in [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) skips the ProxyJump option when `self.domain == target_host.ssh_gateway_domain` — xwing connects directly to aurora.local, which it can since it is the gateway by definition.

### Why Bugs B and C slipped through CI

Both were missed by gaps in test coverage that the existing tests deliberately did not exercise:

- **Bug B**: `tests/test_repository.py` contained only `__str__` tests. No test exercised `Repository.backup()` against more than one host with different `backup_root` values, so the missed `ROOT_DIR` threading never showed up. Added in `lucas42/lucos_backups#227`: `test_backup_uses_each_host_backup_root_in_mkdir`, `test_backup_uses_each_host_backup_root_in_archive_path`, `test_backup_no_hardcoded_root_dir_constant`.
- **Bug C**: `tests/test_shell.py::TestHostOutboundSSH` already had ProxyJump tests covering `avalon._outbound_ssh_args(self.aurora)` (source ≠ gateway), added in v1.0.33 as a regression guard for `lucas42/lucos_backups#160`. They did not cover the source-is-gateway permutation, which is precisely the case that fails. Added in `lucas42/lucos_backups#227`: `test_outbound_ssh_args_skips_proxyjump_when_source_is_gateway`, `test_run_on_remote_no_proxyjump_when_source_is_gateway`.

The wider pattern across all three bugs is the same: **`lucas42/lucos_backups#218`'s design contemplated a single multi-host topology change, but the implementation and its tests treated each host pairing as essentially identical.** The configy null-vs-absent shape (Bug A), per-host parameterisation in repos (Bug B), and the source-is-gateway permutation (Bug C) all exist because the test coverage exercises a single representative host pair rather than the full N×N matrix of source/destination/gateway combinations.

---

## What Was Tried That Didn't Work

- **v1.0.33 (07:14 UTC) was not the fix for Bug A.** It added regression tests for `_outbound_ssh_args` in response to a separate concern (`lucas42/lucos_backups#160`), but did not address the `backup_root` null bug. After this deploy the symptoms persisted unchanged — confirming that whatever was failing was not what v1.0.33 fixed and pointing the investigation at the new aurora code path specifically. As a side-effect of how its tests were scoped, v1.0.33 also did not detect Bug C (the source-is-gateway permutation): its new tests covered `avalon._outbound_ssh_args(aurora)` only.
- **v1.0.34 (10:00 UTC) was not a complete fix either.** It correctly addressed Bug A, restoring `host-tracking-failures` and `volume-host` to green. But because the create-backups cron had previously crashed before reaching the repository loop, Bugs B and C were not visible from CI nor from `/_info` checks, only from running the cron end-to-end. The first ad-hoc rerun at 11:07 UTC was the moment those two bugs became observable.

In hindsight, the lesson is that **a single ad-hoc rerun should be considered the authoritative verification step after any fix to a cron-triggered code path**, not optional. The morning's first ad-hoc rerun was treated as routine confirmation; instead it surfaced two further deterministic bugs that would have re-broken tonight's scheduled cron.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix Bug A: `backup_root` and `shell_flavour` to treat `null` as absent | [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) | Done (merged 2026-04-28) |
| Fix Bugs B + C: thread `backup_root` through `Repository.backup`; skip ProxyJump when source is the gateway. Includes 5 new regression tests covering the missed permutations. | [`lucas42/lucos_backups#227`](https://github.com/lucas42/lucos_backups/pull/227) (closing [`lucas42/lucos_backups#226`](https://github.com/lucas42/lucos_backups/issues/226)) | Done (merged 2026-04-28) |
| Add a dev/prod parity test that loads host config via the configy HTTP API rather than the local YAML, so this class of "absent vs explicit-null" bug is caught in CI | [`lucas42/lucos_backups#223`](https://github.com/lucas42/lucos_backups/issues/223) | Open |
| Emit a `backups` Loganne event with `success=false` (and error detail) when the cron crashes, so detection has a positive failure signal rather than relying on the absence of an expected event. (In this incident the cron crashed before reaching the success-path Loganne emit, leaving silence on the wire — schedule-tracker only saw "expected event missing", which is harder to act on than "this run failed because X".) | [`lucas42/lucos_backups#224`](https://github.com/lucas42/lucos_backups/issues/224) | Open |
| Run the backup cron every 12h with a skip-if-fresh check, so that a missed/crashed run auto-recovers ~12 hours later instead of waiting 24h for the next nightly cycle. (Originally framed in [`lucas42/lucos_schedule_tracker#64`](https://github.com/lucas42/lucos_schedule_tracker/issues/64) as "review the 2-error tolerance"; the architect superseded that approach with a cadence change in `lucos_backups` itself, giving both auto-recovery and faster detection without altering schedule-tracker semantics. `#64` closed as superseded.) | [`lucas42/lucos_backups#225`](https://github.com/lucas42/lucos_backups/issues/225) | Open |
| Investigate whether deploy-window suppression can be narrowed so that `/_info` failures that persist beyond the deploy window are not masked by deploy churn. (In this incident the `host-tracking-failures` and `volume-host` checks were continuously failing from 03:25 UTC onwards, but the post-07:14 deploy of v1.0.33 made the persistent failure visually indistinguishable from the post-deploy flap pattern documented in [`lucas42/lucos_monitoring#186`](https://github.com/lucas42/lucos_monitoring/issues/186) / [`lucas42/lucos_monitoring#195`](https://github.com/lucas42/lucos_monitoring/pull/195).) | [`lucas42/lucos_monitoring#201`](https://github.com/lucas42/lucos_monitoring/issues/201) | Open |
| Codify ad-hoc rerun as the authoritative verification step for any fix to a cron-triggered or scheduled code path, alongside `/_info` and monitoring. (In this incident `/_info` and monitoring went green after the v1.0.34 deploy, but Bugs B and C only became observable when `create-backups` was actually run end-to-end. The SRE persona's "Priority order during incidents" / step 4 should distinguish HTTP-served code paths from cron-triggered ones.) | [`lucas42/lucos_claude_config#50`](https://github.com/lucas42/lucos_claude_config/issues/50) | Open |

---

## Sensitive Findings

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.
