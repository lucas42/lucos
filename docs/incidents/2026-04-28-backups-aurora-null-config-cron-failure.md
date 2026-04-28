# Incident: Backup cron silently failed after Aurora NAS rollout — `dict.get(key, default)` vs configy explicit-null

| Field | Value |
|---|---|
| **Date** | 2026-04-28 |
| **Duration** | ~10 hours, 33 minutes (03:25 UTC missed scheduled run → 14:00 UTC ad-hoc replacement run completed) |
| **Severity** | Data risk — overnight backup cron silently dropped; no `lucos_backups` Loganne event emitted; only the schedule-tracker `lucos_backups_errors=1` count flagged it |
| **Services affected** | `lucos_backups` (cron, `/refresh-tracking`, `/_info` checks `host-tracking-failures` and `volume-host`) |
| **Detected by** | SRE ops check the following morning (no `backups` Loganne event since 2026-04-27 03:58 UTC; schedule-tracker `lucos_backups_errors=1`) |
| **Source issue** | [lucas42/lucos_backups#221](https://github.com/lucas42/lucos_backups/issues/221) |

---

## Summary

The Aurora NAS integration shipped the previous evening (`lucas42/lucos_backups#219`, deployed as v1.0.32 at 23:59 UTC) added `Host` configuration fields — `backup_root` and `shell_flavour` — that are populated only on the new aurora host. For every existing host, `lucos_configy` materialised those absent fields as **explicit `null`** in the JSON response, but the new code used `dict.get(key, default)` which only returns the default when the key is *absent*, not when it is present with a `null` value. As a result, every non-aurora host got `backup_root=None`, the `df -P None …` shell command failed, every host's tracking errored out, the overnight `create-backups` cron crashed without emitting a Loganne event, and the hourly `/refresh-tracking` was failing every run for ~7 hours before the morning ops check caught it. Fixed by replacing `get(key, default)` with `get(key) or default` in [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) and verified end-to-end with an ad-hoc cron run.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-27 18:29 | [`lucas42/lucos#108`](https://github.com/lucas42/lucos/pull/108) (ADR-0001 for Aurora NAS integration) merged |
| 2026-04-27 23:57 | [`lucas42/lucos_backups#219`](https://github.com/lucas42/lucos_backups/pull/219) "Implement aurora NAS integration" merged (commit `1d55b5e`, introduced bug) |
| 2026-04-27 23:59 | `lucos_backups` v1.0.32 deployed to avalon |
| 2026-04-28 03:25 | Scheduled `create-backups` cron run **failed silently** — no `backups` Loganne event emitted |
| 2026-04-28 03:25 → 09:49 | Hourly `/refresh-tracking` cron failing every run; `host-tracking-failures` and `volume-host` checks both `ok=false` |
| 2026-04-28 07:14 | `lucos_backups` v1.0.33 deployed (unrelated fix — added regression tests for `_outbound_ssh_args`; did not address this bug) |
| 2026-04-28 09:49 | SRE ops check opens [`lucas42/lucos_backups#221`](https://github.com/lucas42/lucos_backups/issues/221) (P1) — root cause identified by reproducing inside the production container |
| 2026-04-28 09:58 | [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) merged with the fix |
| 2026-04-28 10:00 | `lucos_backups` v1.0.34 deployed; `host-tracking-failures` and `volume-host` recover; `host-count=4`, `volume-count=34` |
| 2026-04-28 11:07 | Ad-hoc `create-backups` triggered manually (since the scheduled run had been missed) |
| 2026-04-28 ~14:00 | Ad-hoc run completes successfully; `lucos_photos_photos.2026-04-28.tar.gz` and other volumes confirmed on aurora |

---

## Analysis

### The `dict.get(key, default)` / configy explicit-null mismatch

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

The most likely contributing factor: PR #219 was tested locally against the YAML file at `~/sandboxes/lucos_configy/config/hosts.yaml`, where optional fields are simply omitted from the host record. That shape behaves correctly with `dict.get(key, default)`. The configy HTTP API normalises the response to include every defined field (with `null` for unset ones), and that shape is what production sees. There was no test exercising the configy API shape directly, only the underlying YAML.

This is a recurring class of dev/prod parity issue with configy that is worth eliminating with a fixture / contract test.

### Why detection took ~7 hours

The cron job at 03:25 UTC failed before reaching the point that publishes the `backups` Loganne event, so no monitoring webhook fired. The schedule-tracker check on `lucos_backups` tolerates 2 consecutive errors before flagging — last night's run incremented the count from 0 to 1, which does not fire an alert. The `/_info` endpoint reported `host-tracking-failures: ok=false` and `volume-host: ok=false` continuously from ~03:25 onwards, which monitoring **did** see — but those particular checks were already flapping during the deploy wave (suppressed during the deploy window), so the post-suppression alert was indistinguishable from the noise around the v1.0.32 → v1.0.33 deploy churn until the morning ops check sat down with the actual `/_info` body and dug in.

---

## What Was Tried That Didn't Work

- **v1.0.33 (07:14 UTC) was not the fix.** It added regression tests for `_outbound_ssh_args` in response to a separate concern, but did not address the `backup_root` null bug. After this deploy the symptoms persisted unchanged — confirming that whatever was failing was not what v1.0.33 fixed and pointing the investigation at the new aurora code path specifically.

The actual fix landed first time. No other dead-ends.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix `backup_root` and `shell_flavour` to treat `null` as absent | [`lucas42/lucos_backups#222`](https://github.com/lucas42/lucos_backups/pull/222) | Done (merged 2026-04-28) |
| Add a dev/prod parity test that loads host config via the configy HTTP API rather than the local YAML, so this class of "absent vs explicit-null" bug is caught in CI | Not yet tracked — will raise after this report merges | Open |

---

## Sensitive Findings

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.
