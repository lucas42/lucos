# Incident: lucos_creds blocked by stale CircleCI deploy snapshot masking corrupted SSH keys

| Field | Value |
|---|---|
| **Date** | 2026-05-09 |
| **Duration** | ~1h 55m (11:55 UTC to 13:50 UTC) |
| **Severity** | Partial degradation — `lucos_creds_ui` unable to authenticate to creds backend; `lucos_creds_configy_sync` unable to push credential updates to consumer services |
| **Services affected** | `lucos_creds` (UI + configy_sync), `lucos_schedule_tracker` (cascading health check) |
| **Detected by** | monitoring alert (`fetch-info` on creds.l42.eu, `lucos_docker_health_avalon` on schedule-tracker.l42.eu) |

---

## Summary

PR [lucas42/lucos_creds#303](https://github.com/lucas42/lucos_creds/pull/303) removed a long-standing character-substitution workaround from the SSH-key startup paths in `lucos_creds_ui` and `lucos_creds_configy_sync`. The new code writes `UI_PRIVATE_SSH_KEY` and `CONFIGY_SYNC_PRIVATE_SSH_KEY` directly to disk without any transformation. When the production credentials in lucos_creds storage contained CRLF (`\r\n`) line endings, the resulting `id_ed25519` files passed OpenSSH's PEM-wrapper line-by-line parsing but failed `libcrypto`'s base64 decode of the key body — `\r` is not in the base64 alphabet — and SSH authentication failed.

The actual recovery was complicated by a second factor: lucos_creds uses a self-deploy mechanism (PR [lucas42/lucos_creds#233](https://github.com/lucas42/lucos_creds/pull/233), closing [lucas42/lucos_creds#152](https://github.com/lucas42/lucos_creds/issues/152)) that reads `.env` from a CircleCI project env var `LUCOS_DEPLOY_ENV_BASE64` rather than SCP'ing from creds.l42.eu, to break the circular deploy dependency that bit a previous incident on 2026-03-26/27. That env var holds a base64-encoded `.env` snapshot — set on 2026-04-10 — which had **never been refreshed** when production credentials changed. So when lucas42 corrected the keys in lucos_creds storage and triggered a redeploy, the deploy orb decoded the stale base64 blob and overwrote the new container's `.env` with the original CRLF-corrupted credentials. The first redeploy attempt (pipeline 680) failed for exactly this reason. Resolution required updating both lucos_creds storage **and** the `LUCOS_DEPLOY_ENV_BASE64` CircleCI env var, then redeploying (pipeline 682).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-10 | `LUCOS_DEPLOY_ENV_BASE64` set in CircleCI as part of breaking the circular deploy dependency. Snapshot of production `.env` at that moment, including the CRLF-corrupted SSH keys |
| 11:50:43 | `UI_PRIVATE_SSH_KEY` re-stored in `development` (lucas42's pre-merge prep) |
| 11:51:51 | `CONFIGY_SYNC_PRIVATE_SSH_KEY` re-stored in `development` and `production` |
| 11:52:27 | `UI_PRIVATE_SSH_KEY` re-stored in `production` |
| 11:55:50 | PR #303 merged to `main` (commit `1f8b388`) |
| 11:55:52 | CircleCI pipeline 678 created (auto-deploy on merge) |
| ~11:57 | `lucos_creds v1.0.41` deployed to avalon. Deploy orb writes `.env` from the stale `LUCOS_DEPLOY_ENV_BASE64` snapshot. New code writes the corrupted env-var bytes verbatim to `id_ed25519`. Both `lucos_creds_ui` and `lucos_creds_configy_sync` fail SSH on first request |
| 12:01:00 | First `monitoringAlertSuppressed` for creds.l42.eu `fetch-info` (suppressed during deploy window) |
| 12:04:07 | Suppressed alert extends to `circleci` + `fetch-info` |
| 12:08:15 | `monitoringAlert` on schedule-tracker — `lucos_docker_health_avalon` failing because docker reports `lucos_creds_ui` as unhealthy |
| 13:04:20 | `monitoringAlert` extends to also include the `lucos_creds_configy_sync` schedule check |
| ~13:15 | lucas42 inspects the stored credentials in lucos_creds, identifies CRLF as the corruption, and re-stores both keys in the live store with LF-only line endings |
| 13:19:42 | `CONFIGY_SYNC_PRIVATE_SSH_KEY` re-stored with LF-only |
| 13:21:05 | `UI_PRIVATE_SSH_KEY` re-stored with LF-only |
| 13:21:57 | CircleCI pipeline 680 triggered via API (manual redeploy on same SHA) — but the deploy still reads `LUCOS_DEPLOY_ENV_BASE64`, which is unchanged, so the corrupted keys are written again |
| 13:26:21 | Brief `monitoringRecovery` on schedule-tracker as the old container is torn down before the new one fails its healthcheck — this clears the docker_health alert temporarily |
| 13:28:22 | Pipeline 680 fails: `docker compose up --wait` reports `lucos_creds_ui` unhealthy after retries; orb's deploy retry loop bails after `MAX_ATTEMPTS=3` |
| 13:33:22 | `monitoringAlert` re-fires for `lucos_docker_health_avalon` once the failed-deploy state stabilises |
| ~13:40 | lucas42 (with team-lead's input) identifies that the deploy is reading a stale snapshot from `LUCOS_DEPLOY_ENV_BASE64`, not the live store. Updates the CircleCI env var with the corrected base64 of the post-fix `.env` |
| 13:46:29 | CircleCI pipeline 682 triggered via API — this time decoding the corrected snapshot |
| 13:48:50 | `lucos/deploy-avalon` job starts; new containers receive correct keys |
| 13:50:17 | `lucos_creds v1.0.44` deployed successfully — pipeline 682 succeeds |
| 13:50:24 | `monitoringRecovery` — all schedule-tracker checks healthy |
| ~13:50 | New `lucos_creds_configy_sync` container's startup `sync.py` invocation runs end-to-end (SSH to backend → fetch from configy.l42.eu → write back to creds → POST `status: success` to schedule-tracker) |
| 13:53:00 | First post-fix scheduled `:53` cron tick of `configy_sync` runs (additional positive evidence) |
| ~13:53 | `creds.l42.eu/_info`: `ssh-server: ok=true`, `metrics.systems.value: 50` (UI is reading credentials end-to-end via SSH from the backend, listing 50 systems) — full recovery confirmed |

---

## Analysis

### Stage 1 — CRLF corruption in stored SSH keys

`UI_PRIVATE_SSH_KEY` and `CONFIGY_SYNC_PRIVATE_SSH_KEY` had been stored in lucos_creds with CRLF line endings between PEM lines. Before PR #303 the startup code path passed the env var through a substitution pipeline (`~` → `=`, literal `\n` → real newline) that was written to undo the encoding required before lucos_creds supported multiline `.env` delivery. Whatever combination of character classes the OLD pipeline tolerated, the corruption stayed latent — not because the key file was clean, but because the key was working "well enough" for the limited operations involved.

PR #303 removed the pipeline. The new code (`fs.writeFileSync('/root/.ssh/id_ed25519', process.env.UI_PRIVATE_SSH_KEY)` for the UI, `printf '%s' "$CONFIGY_SYNC_PRIVATE_SSH_KEY" > /root/.ssh/id_ed25519` for configy_sync) writes whatever bytes are in the env var directly to disk. With CRLF in the env var, the resulting key file passes OpenSSH's PEM-wrapper line-by-line parsing but fails libcrypto's base64 decode of the key body — `\r` is not in the base64 alphabet. The error surfaces as:

```
Load key "/root/.ssh/id_ed25519": error in libcrypto
```

This message is OpenSSH's catch-all for "PEM wrapper looked plausible, key body did not decode," and gives no hint that line endings are the culprit.

### Stage 2 — Stale CircleCI snapshot perpetuating the corruption

lucos_creds cannot SCP its own `.env` from itself during deploy without a circular dependency (the deploy orb's default path is `creds.l42.eu:2202` — which is the very service being redeployed). PR #233 (closing #152) introduced an alternative: a CircleCI project env var `LUCOS_DEPLOY_ENV_BASE64` containing a base64-encoded snapshot of the production `.env`. When this var is set, the deploy orb decodes it directly into `.env` on the deploy target, skipping SCP entirely and skipping the host-key-scan against creds.l42.eu.

This solves the bootstrap problem but introduces an out-of-band mirror of the production credential set. Updates to lucos_creds storage **do not propagate** to the snapshot; the snapshot is updated only by an explicit manual step (re-encoding the current `.env` and `PUT`ing it via the CircleCI API).

`LUCOS_DEPLOY_ENV_BASE64` was last set on 2026-04-10. Since then there have been changes to lucos_creds storage (including, by the time of this incident, the CRLF re-stores and the LF-only re-stores) — but the snapshot was never updated to match. So every redeploy of lucos_creds — including the auto-deploy on PR #303's merge, the manual redeploy after lucas42 fixed the live store (pipeline 680), and any other redeploy in the intervening month — wrote the **same** 2026-04-10 snapshot to disk, corrupted SSH keys included.

### Stage 3 — Misleading early signals during diagnosis

Two signals during diagnosis suggested progress that wasn't actually progress, and led the on-call SRE (me) to a confidently-wrong report:

1. **The `lucos_creds_configy_sync` schedule-tracker alert cleared between 13:04:20Z and 13:26:21Z**, despite no working-key path existing during that window. The most plausible explanation is that schedule-tracker's check definition is "any of the 3 most recently finished runs were successful AND most recent within 10800s" — a 3-hour positive window. A successful cron run from BEFORE the deploy at 11:55Z could continue to satisfy "most recent within 3h" until ~14:55Z. Once additional failed runs accumulated and the older success aged out, the alert fired again. The alert *clearing* was therefore a stale-positive artefact, not evidence of a fix taking effect.
2. **`lucos_creds_configy_sync` reached docker `Healthy` in 2.0s during pipeline 680's deploy attempt.** This was easy to read as "configy_sync's key works." It does not. configy_sync's docker healthcheck is `test -p /var/log/cron.log` — it only verifies that the cron daemon initialised the named pipe, which happens regardless of whether the SSH key is valid. The actual sync run is a separate process that exits non-zero on SSH failure, but the container is already marked healthy by then.

I sent an intermediate report to team-lead at 13:32Z attributing the failed redeploy to "lucas42's UI re-store didn't take." That was wrong — the UI re-store had taken in lucos_creds storage exactly as he intended; the deploy orb was overwriting it from the stale snapshot. The right next diagnostic question would have been "does the deploy actually read from the current store, or from a snapshot?" — and the answer is documented in PR #233, which I had not consulted.

---

## What Was Tried That Didn't Work

1. **Initial SRE diagnosis: literal `\n` strings or `~`-for-`=` substitution still in the stored value.** Same shape (env var on disk, libcrypto rejects base64 body), wrong specific characters. Easy to over-fit `Load key … error in libcrypto` to whichever non-base64 characters you've recently been thinking about. Future responders should treat that error as "any of: `\r`, literal `\n`, `~`-substitution, leading/trailing whitespace, BOM, etc." and inspect the file bytes directly.
2. **Pipeline 680: redeploy after fixing live store only.** Failed because `LUCOS_DEPLOY_ENV_BASE64` is not connected to the live store; the deploy decoded the same stale snapshot and wrote the same broken file.
3. **Considering rollback to v1.0.39.** Rejected. Rolling back to the pre-PR code would only have helped if the credential were in the *old* substituted format, but lucas42 had moved both stored values to the post-substitution shape — and crucially, the deploy orb would have written the stale snapshot regardless of which application code it was invoking. Rollback would not have addressed the snapshot problem and would have re-introduced the workaround path against credentials that no longer needed it. Forward through both the credential rewrite and the snapshot rewrite was the only viable direction.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Document the dual-update requirement for production credentials in lucos_creds (live store **and** `LUCOS_DEPLOY_ENV_BASE64`); add to operational runbook | lucas42/lucos_creds#304 | Open |
| In-product warning in lucos_creds_ui at credential-edit time when serving its own production credentials (belt-and-braces over the README/runbook fix) | lucas42/lucos_creds#305 | Open |
| Validate `UI_PRIVATE_SSH_KEY` / `CONFIGY_SYNC_PRIVATE_SSH_KEY` at container startup before writing to disk — reject CRLF, `~`-substitution, literal `\n`, missing PEM wrapper. Surfaces the failure at startup with a clear human-readable error rather than as a libcrypto puzzle | lucas42/lucos_creds#306 | Open |
| Improve the `Load key: error in libcrypto` diagnostic in the `ssh-server` `/_info` check (e.g. emit the first 16 bytes of the body in escaped form so responders can spot CRLF / BOM / `~` at a glance without filesystem access) | Considered but not filed — `lucas42/lucos_creds#306` (startup-time validation) prevents the entire failure class earlier and with a clearer error message, making the runtime diagnostic improvement largely redundant per the follow-up calibration rule | Not filed |
| Consider auto-syncing `LUCOS_DEPLOY_ENV_BASE64` from the live store as part of any credential change touching lucos_creds itself (architectural — needs lucos-architect input on how to do this without re-introducing the circular dependency from #152). lucas42 has indicated 2026-05-09 that the structural fix is not affordable right now, so any future architect consultation should ask "is there a low-cost variant, or should this be closed `not_planned`?" rather than treat it as fresh planning | TBD | Not yet filed |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.

The corrupted SSH private keys never authenticated successfully (libcrypto rejected them outright), so no unauthorised SSH session was established. The actual key material is unchanged from before the incident — the rotation in this report is purely a re-encoding from CRLF to LF, not a re-keying. No key bytes appear in this report.
