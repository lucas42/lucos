# Incident: aithne KEK-derivation change auto-deployed before its one-time migration

| Field | Value |
|---|---|
| **Date** | 2026-06-30 |
| **Duration** | ~46 minutes (15:51 UTC to 16:37 UTC) |
| **Severity** | Complete outage (estate-wide authentication — new login / token issuance) |
| **Services affected** | `lucos_aithne` (estate auth SPOF); by extension, every consumer's ability to mint *new* sessions |
| **Detected by** | Identified during the coordinated KEK-migration handoff (team-lead dispatch) — aithne was found already crash-looping |

Originating change: lucas42/lucos_aithne#260 (Closes lucas42/lucos_aithne#244).

---

## Summary

A change to how `lucos_aithne` derives its signing-key encryption key (`SIGNING_KEK`) — from raw bytes to `sha256(value)` — was merged (lucas42/lucos_aithne#260, released as v2.0.0). The change is correct, but it requires a **one-time `--migrate-kek`** to be run against the production credential store *before* the new image serves, because the existing signing key is wrapped under the legacy raw-bytes derivation. lucos auto-deploys on merge, so the new image deployed and started **before** the migration was run; it could not decrypt the signing key and **crash-looped**, taking out new-token issuance and login estate-wide for ~46 minutes. Recovery was the intended path executed under control: stop the container, snapshot the volume, run `--migrate-kek` to re-wrap the signing key under a fresh 256-bit KEK, deploy the new KEK, and redeploy. The signing key itself was **preserved** through the migration (not regenerated), so existing sessions were never actually invalidated.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 15:48:54 | lucas42/lucos_aithne#260 merged to main (SHA-256 `SIGNING_KEK` derivation; semantic-release → v2.0.0). |
| 15:51:08 | CI auto-deploy recreated the container on v2.0.0. The startup signing-key migration tries to AES-GCM-decrypt the stored key with the new sha256-derived KEK; the key is wrapped under the legacy raw-bytes KEK, so it fails with `…neither AES-GCM ciphertext nor valid PKCS8 DER — database may be corrupt`. Container begins crash-looping. **Estate-wide new-token issuance / login down.** |
| ~16:05 | SRE engaged (team-lead dispatch to own the migration). aithne found already crash-looping; root cause confirmed from container logs. |
| 16:13 | SRE stopped the crash-looping container and took a verified pre-migration snapshot of the `lucos_aithne_credential_store` volume (rollback point). |
| ~16:25 | Coordinated with lucas42: chose a **fresh 256-bit `SIGNING_KEK`** (the full intent of #244). New value staged through lucos_creds *development* (a key the agent can read) to keep the production secret out of the message channel. |
| 16:33:22 | `--migrate-kek` succeeded — re-encrypted 1 signing key under the new SHA-256-derived KEK. (Two earlier attempts failed *before any DB write* — see "What Was Tried That Didn't Work".) |
| ~16:34 | Temp dev handoff key cleared from creds; fresh CircleCI deploy triggered (pipeline #442) to recreate the container with the new `SIGNING_KEK` from production creds. |
| 16:37:04 | Container recreated on v2.0.1, **healthy**. `/_info` green (`db`, `signing_key`, `signing_key_age`); JWKS serving the **preserved** signing key (kid `15afb754…`). **Service restored.** |
| (post-recovery) | Human passkey login at `aithne.l42.eu/auth/login` confirmed working (lucas42). Recovery verified end-to-end across both the machine path (JWKS / token issuance) and the human login path. |

---

## Analysis

### Root cause — a breaking one-time migration raced its own auto-deploy

lucos_aithne#260 changed the `SIGNING_KEK` contract: the env value is now run through `sha256.Sum256()` to derive the AES-256-GCM key, instead of being consumed as raw bytes. Existing signing keys on disk are AES-GCM ciphertext under the *old* (raw-bytes) derivation, so the new image must re-wrap them — that is what the one-time `--migrate-kek` subcommand does — **before** it tries to open them normally.

lucos deploys automatically on merge to `main`. There was nothing holding the deploy until the migration had run, so the sequence was: merge → build → deploy → new image starts → startup `MigrateSigningKeyEncryption` can't decrypt the signing key under the new KEK → `log.Fatal` → restart → repeat. The migration is a manual, out-of-band operator action, and the release shipped with no gate coupling the two. The race is the root cause; it is a property of "breaking on-disk migration" + "auto-deploy on merge" with no coordination between them.

The documented procedure tacitly assumed a human in the loop between merge and restart, in which the operator runs `--migrate-kek` before bringing the new image up. **Auto-merge plus auto-deploy closes that window to zero** — there is no human-occupied gap to run the migration in. A migration-coupled release therefore cannot rely on operator timing; the coupling has to be enforced by the system (gate or fail-soft startup), which is what lucas42/lucos_aithne#261 tracks.

### Aggravating factor — the failure presents as "database may be corrupt"

The startup migration cannot distinguish "AES-GCM ciphertext under a *different* KEK" from "corrupt / not ciphertext at all", so it reports `database may be corrupt`. The data was not corrupt — it was correctly-wrapped ciphertext under the old KEK derivation. A responder taking that message at face value could be steered toward a *data-recovery* path (restore from backup, suspect disk corruption) instead of the actual fix (run the KEK migration). Clearer diagnostics here would shorten any future occurrence.

### Recovery friction — the documented procedure didn't match the running system

The `--migrate-kek` procedure in the PR body referenced `lucos_aithne_web` (a stale container/image name; the real names are `lucos_aithne` / `lucas42/lucos_aithne` / `lucos_aithne_credential_store`) and assumed a Docker entrypoint that the image does not set. In addition, `main()` calls `getEnvRequired("PORT")` *before* dispatching to any subcommand, so `--migrate-kek` needs `PORT` set even though it never uses it; and the creds-emitted `.env` wraps every value in double quotes, which the deploy strips before the container sees them. None of these caused damage — each surfaced as a clean non-zero exit *before* any database write — but together they cost time during an active outage.

**Near miss worth recording (flagged by security):** the double-quoting was not just friction — it was a trap for a *silent* failure. Had the quoted 46-char value been fed to `--migrate-kek` verbatim, the migration would have exited **0** (it would have happily re-wrapped the key under `sha256("…46-char…")`), but the redeployed container — reading the *unquoted* 44-char value from creds — would compute `sha256("…44-char…")` and crash-loop again. The symptom would then have been the maddening "we ran the migration and it succeeded, why is it still crashing?" An off-by-quote on the KEK produces *migrate-success-then-startup-crash*, not a clean error. This is why the value fed to `--migrate-kek` must be the exact bytes the container will receive (unquoted), and it is called out in lucas42/lucos_aithne#262 as a documentation requirement.

---

## What Was Tried That Didn't Work

- **`docker run … lucas42/lucos_aithne:2.0.0 --migrate-kek`** → `exec: "--migrate-kek": executable file not found`. The image has no entrypoint; the binary is `/lucos_aithne`. Fix: `… /lucos_aithne --migrate-kek`.
- **`… /lucos_aithne --migrate-kek` with only `SIGNING_KEK`/`NEW_SIGNING_KEK` set** → `Required environment variable PORT is not set`. `main()` validates `PORT` before subcommand dispatch. Fix: pass any `PORT`.
- **Using the staged KEK value verbatim** (46 chars) → it was double-quoted in the creds `.env`; the deploy strips the quotes before the container reads it (proven: dev `SIGNING_KEK` is 34 chars quoted but the container env held it at 32). The value fed to `--migrate-kek` had to be the **unquoted 44-char** form so `sha256(new)` matches what the recreated container computes.
- **`docker start` of the stopped container** was considered and rejected: it reuses the container's baked-in env (the *old* `SIGNING_KEK`), so it would have crash-looped again. The container had to be **recreated** via a fresh deploy so it reads the new `SIGNING_KEK` from production creds.

All three of the genuine errors above were caught before any irreversible change; the only DB-mutating step (`--migrate-kek`) ran exactly once, successfully. A verified pre-migration volume snapshot was held as a rollback point throughout and was not needed.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Couple breaking on-disk migrations to deployment so an auto-deploy can't land a release before its required migration has run (deploy gate, or startup that migrates/halts-cleanly instead of crash-looping) — needs architect input | lucas42/lucos_aithne#261 | Open |
| aithne post-migration hardening: clearer "signing key undecryptable under current SIGNING_KEK — run `--migrate-kek`?" message instead of "database may be corrupt"; drop the `PORT` requirement for maintenance subcommands; fix the `--migrate-kek`/`--rekey` procedure docs (real container/image/volume names, `/lucos_aithne` invocation, byte-exact unquoted KEK, redeploy-not-`docker start`) | lucas42/lucos_aithne#262 | Open |
| User-facing auth-failure messaging: what consumers display when aithne is unavailable (raw 500 vs a meaningful "sign-in temporarily unavailable" state). A pre-existing latent concern surfaced by this incident, not a direct corrective — owned by lucos-ux per the incident-notification coordination | lucas42/lucos_ux#TBD (lucos-ux filing) | Open |
| Remove the ad-hoc pre-migration snapshot `lucos_aithne_credential_store.pre-migrate-kek-2026-06-30.tar.gz` from `avalon:/srv/backups/local/volume/` once the report is finalised (non-standard filename; kept as rollback during the incident) | — | Open |

---

## Sensitive Findings

[x] Yes — see note below.

The incident centred on the `SIGNING_KEK` (the AES-256-GCM key-encryption key that wraps aithne's signing keys). No key material is included in this report. During recovery, key values were handled without printing them (lengths only were logged for sanity checks), the new production value was staged through lucos_creds *development* rather than the message channel, and the temporary staging key was deleted after consumption. The production `SIGNING_KEK` was written to creds by lucas42 (agents cannot write production creds). The signing key itself was preserved through the migration; only its at-rest wrapping changed.
