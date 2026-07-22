# Incident: configy_sync SSH key corrupted across two rounds, masked by a false-green dashboard

| Field | Value |
|---|---|
| **Date** | 2026-07-19 |
| **Duration** | ~62 hours (2026-07-19 10:11 UTC to 2026-07-22 00:04 UTC) |
| **Severity** | Partial degradation (config sync stalled; no direct user-facing impact) |
| **Services affected** | `lucos_creds` (configy_sync sync stalled; deploy path blocked, then silently broken); `lucos_docker_health` (avalon check); blocked deploys incl. ADR-0005 (`lucas42/lucos_creds#472`) |
| **Detected by** | Monitoring alerts (caught the ~60h crash-loop throughout); the ~69-min Round-2 false-recovery was found by manual end-to-end verification, not monitoring |

Source issue: `lucas42/lucos_creds#474`.

---

## Summary

A routine rotation of the `lucos_creds_configy_sync` SSH keypair (`lucas42/lucos_creds#471`) exposed that the production `CONFIGY_SYNC_PRIVATE_SSH_KEY` had been stored corrupted, breaking config sync from `lucos_configy` into the credential store for ~62 hours. The corruption happened twice, in two different ways: first CRLF line-endings (introduced when the key was saved through the credential UI), which tripped the container's CR-guard and **crash-looped it for ~60 hours** — loudly, with monitoring correctly showing red; then, during the first fix attempt, a stripped trailing newline (bash `$(...)` command substitution), producing a full-length-but-cryptographically-invalid key. The second round was the insidious one: the invalid key cleared the startup guard, so the container ran "healthy" and monitoring **falsely recovered to all-green while sync was still completely broken** — a ~69-minute window before manual verification caught it. (Monitoring correctly caught the ~60h crash-loop throughout; the gap was this false *recovery*, not a blind spot.) It was resolved by producing and validating a known-good key in dev (root-caused by the sysadmin), copying the exact bytes into the prod store with a byte-safe write, and redeploying — verified end-to-end by a real sync completing, not by container health.

---

## Timeline

All times UTC. Anchored to GitHub comment timestamps, CircleCI pipeline times, and container `StartedAt` — the responder's local clock skewed ~1.5h during the incident and is not used.

| Time (UTC) | Event |
|---|---|
| 2026-05-11 | `configy_sync/startup.sh` gains a CR-guard that fail-closes (exit 1) if the key contains a CR. Passes cleanly for two months. |
| 2026-07-19 10:08:35 | `lucas42/lucos_creds#471` merges (regenerate a distinct configy_sync keypair), deploy-on-merge. Diff is one line in `authorized_keys` and is correct. |
| 2026-07-19 10:11:04 | Deploy (pipeline 1085, `deploy-avalon`) recreates configy_sync; it re-reads the stored key — CRLF-corrupted — and **crash-loops on the CR-guard** (the guard fires on the carriage returns). Deploy job fails; `lucos_creds` circleci + `lucos_docker_health` avalon checks go red. |
| 2026-07-19 10:43 | Incident raised as `lucas42/lucos_creds#474`. |
| 2026-07-19 10:45 | Triaged Ready / Owner lucas42 / High (fix is a production credential write; agents cannot write prod creds). |
| 2026-07-19 → 21 | Crash-loop continues (~1000+ restarts, ~60h). **Monitoring correctly catches it the whole time** (per the loganne record): the `configy_sync` check fires `monitoringAlert`s repeatedly (07-19 12:53, 07-20 07:13–07:38 & 22:30, 07-21 07:11–07:28) and `lucos_docker_health/avalon` flaps red↔green hourly. The dashboard shows the incident throughout this phase. |
| 2026-07-21 22:54:39 | First fix attempt deploys as `v1.3.81` (pipeline 1108-predecessor). The re-stored key is a **new** corruption — full-length but invalid (trailing newline stripped). The invalid key clears the CR-guard, so the container stops crash-looping (StartedAt 22:54:18) and goes docker-`healthy`; the resumed-but-failing sync run reports its first error. |
| 2026-07-21 22:55 | **False *recovery*.** The `configy_sync` check fires `monitoringRecovery` ("all checks healthy") at 22:55:39 even though every sync run auth-fails — the genuine gap, a ~69-min window (mechanism in Analysis; `lucas42/lucos_schedule_tracker#96`). (`docker_health/avalon` also recovered at 22:55:04 because the container was now docker-healthy — by design; not a defect.) |
| 2026-07-21 23:02 | Responder identifies the false recovery from inside the container (`ssh-keygen` → "not a key file"; `error in libcrypto`; sync failing) and reports it NOT resolved. |
| 2026-07-21 23:47 | Sysadmin (independently, during separate dev-environment work) root-causes the trailing-newline strip in the dev store, validates a known-good key end-to-end, and posts a byte-safe write recipe. |
| 2026-07-21 23:59:44 | lucas42 copies the validated key bytes into the prod store (byte-safe, no `$(...)` capture) and retriggers the deploy (pipeline 1108). |
| 2026-07-22 00:04:23 | Real fix deploys as `v1.3.82`; container recreates with the valid key (StartedAt 00:04:14). Startup sync run: `Sync Complete` (00:04:31). Ends the ~69-min false-recovery window. |
| 2026-07-22 00:04–00:06 | Verified end-to-end: in-container key valid and fingerprint matches the dev-validated one; an ad-hoc idempotent re-run also completes; both monitoring reds clear. **Resolved.** |

---

## Analysis

This incident cascaded through several conditions. No single one was "the" cause; each made the incident possible, worse, or slower to detect.

### Round 1 — CRLF corruption (the original outage)

The production `CONFIGY_SYNC_PRIVATE_SSH_KEY` was stored with CRLF line endings. As diagnosed in `lucas42/lucos_creds#476`, browsers CRLF-normalise `<textarea>` content on native form submission (a WHATWG spec behaviour, not a browser bug), and the credential UI's POST handler passed the value to storage without normalising. The Go SSH server stores whatever follows the first `=` verbatim, so the CRLF value was stored *whole*. This write-path defect has since been fixed on `main` in commit `3ce4817` (closing `#476`; it superseded the earlier PR `#477`, which was closed unmerged).

The crash-loop follows directly from that: the `startup.sh` CR-guard fires on *any* carriage return in the value (`case "$KEY" in *$'\r'*`), so the CRLF-corrupted key trips it and the container exits 1 on every start, fail-closing rather than writing a broken key to disk. `lucas42/lucos_creds#471`'s rotation didn't *cause* the corruption — it made a rotation necessary, and the value written by that rotation was already malformed. The guard doing its job is why this surfaced as a loud crash-loop (monitoring red) instead of a silent auth failure.

(An earlier draft of this report attributed the crash-loop to the deploy's `.env` parser truncating the multi-line value at its first newline. That was an unverified hypothesis — carried from a `#476` discussion where it was explicitly hedged — and it rested on a "36-char" measurement that later proved to be a `docker inspect` artifact, not a real truncation, see below. The CR-guard firing on the whole CRLF value is sufficient and verified; no truncation is needed to explain Round 1, and the truncation claim has been removed.)

### Round 2 — trailing-newline strip (the fix that introduced a second corruption)

The first fix attempt re-stored the key and converted line endings, but produced a **different** corruption: a full-length, LF-only value with correct `BEGIN`/`END` framing and no CRs that nonetheless failed `ssh-keygen -l -f` ("not a key file") and failed SSH client load with `error in libcrypto`. Root cause (found by the sysadmin in the dev store): **bash `$(...)` command substitution strips all trailing newlines from the captured value.** An OpenSSH key ends `-----END OPENSSH PRIVATE KEY-----\n`; that final newline silently vanished, leaving a key one byte short and structurally invalid. The fixed key is 419 bytes on disk vs the broken 418 — that recovered byte is the restored trailing newline, confirming the mechanism from both directions.

### Delayed detection — a false *recovery*, not a blind spot (~69 minutes)

Monitoring **caught** this outage — a claim worth stating carefully, because an earlier draft (and the first version of the follow-up ticket) got it badly wrong. The loganne `monitoringAlert`/`monitoringRecovery` record is authoritative, and it shows the `configy_sync` check firing alerts repeatedly across 07-19/20/21, plus `lucos_docker_health/avalon` flapping red hourly. The dashboard was **red**, correctly, for the whole ~60h crash-loop.

The genuine gap is a **~69-minute false *recovery***: at 22:55 the `configy_sync` check fired `monitoringRecovery` ("all healthy") right after the `v1.3.81` invalid-key deploy, and stayed green until the real fix `v1.3.82` at 00:04 — while sync auth-failed every run.

The mechanism is a specific bug in `schedule_tracker/src/database.rb`, established from genuine logs plus source (not reconstruction). A check is green iff `error_count < threshold` **and** `age < time_threshold`, where `age` is computed from `last_success || last_error`. `updateScheduleError` writes via `INSERT OR REPLACE` **omitting `last_success`**, so every error report nulls `last_success` — and `age` then falls back to the most recent *error*. The proof:

- Throughout the crash-loop, the `configy_sync` alerts' own `failingChecks` debug reads `"Job last ran at 2026-07-19T09:53:15, which is N seconds ago (threshold 10800s)"`, N growing 78k→131k→163k→219k — so `age` was anchored on `last_success = 09:53:15`, which never advanced (no success recorded), and the check correctly red via the **age branch**.
- At 22:55 the check recovered — impossible if `age` were still anchored on `09:53:15` (~220000s → red). So `age` fell back to `last_error`, which happens only if `last_success` was nulled. The first error report after the crash-loop did exactly that; `error_count` was 1 (`< 3`) → green.

So **reporting an error erases when the job last succeeded**, and a job that resumes running but keeps failing recovers its own check on the first error. The staleness clause guards against "stopped running", not "running-but-failing". (This corrects an earlier draft's "fail-open / silently-dead job reads green" framing — it does *not*; a stopped job reds via the age branch, as the debug above shows. lucas42 flagged that error; the gap is specifically the recovery edge.)

`lucos_docker_health/avalon` also recovered at 22:55, because the invalid-key container was docker-`healthy`. That is **by design** — the container healthcheck checks liveness, not work-success, and isn't meant to understand the latter — so it's noted as context, not a defect.

The `database.rb` bug is tracked in `lucas42/lucos_schedule_tracker#96` (re-scoped from the original "estate-wide fail-open blind spot" thesis to this concrete, source-verified bug), pending lucas42's confirmation that the `INSERT OR REPLACE` behaviour is unintended.

### Underlying enabler — no write-time validation, and a line-oriented transport carrying multi-line secrets

Both corruptions share a signature: the store accepted a malformed value silently, and the damage surfaced elsewhere, later. Two structural conditions underlie that, and — per architectural review — they are **complementary, not the same problem**:

- **No write-time validation.** The store validates neither line-endings nor value content on write, so a bad value is accepted and only detected downstream. A cheap, type-agnostic reject of bare CR / control characters at the write (`lucas42/lucos_creds#473`) would have caught Round 1 there. *Typed-value* validation (decode-then-`ssh-keygen`) would catch Round 2, but presupposes a value-typing model the store doesn't yet have — a real design decision, tracked separately (see Follow-up Actions).

- **A line-oriented transport carrying inherently multi-line secrets.** The store→shell→container→disk path is line-oriented, but SSH/PEM keys are multi-line: a category mismatch that is the common root of all three corruption modes (CRLF, header-truncation, trailing-newline-strip). The response so far has been to accrete *destination-side* guards — `configy_sync/startup.sh`'s `case` (CR, `~`, PEM-header) and the creds UI's `validateSshKey()`. Per architectural review (`lucas42/lucos_creds#484`), these have **already drifted apart** — the UI checks a PEM footer the shell `case` doesn't — and, more damningly, **both are blind to Round 2 by construction**: a full-length, LF-only, correctly-framed but one-byte-short key passes *all* of their checks across the two consumers. Every guard is a *detector* that fires *after* transport, not a fix that prevents corruption; Round 1 tripped the CR guard (loud, good), Round 2 cleared everything (silent, bad). A growing pile of divergent destination guards that still can't catch a one-byte-short key is the smell that the class is being fought where it's *observed*, not where it *happens*.

The robust direction (architectural follow-up below) is to make key-typed values **single-line for the whole transport** — base64 at rest, decode-and-validate at point of use — which dissolves the transport class at once (estate precedent: `LUCOS_DEPLOY_ENV_BASE64` base64s the entire `.env` for exactly this reason; there are already ≥2 multi-line keys on this path — `CONFIGY_SYNC_PRIVATE_SSH_KEY` and `UI_PRIVATE_SSH_KEY`). Crucially, write-validation and transport-hardening close *different* holes — a correct multi-line key still can't traverse a line-oriented transport intact, and a garbage-that-decodes-to-non-key still stores — so neither alone closes the class.

### What slowed resolution — the PR #477 review churn

The incident took ~62 hours to resolve, and a chunk of that was not the credential fix itself but churn around the *code* fix for the underlying UI defect. PR `lucas42/lucos_creds#477` (the `normalizeLineEndings` write-path fix for `#476`) ran **six review rounds** across 07-20 — successive commits (`584cd20` → `927b328` → `5709f56` → `654ad4e` → `c2077fd` → `f44b877`), each addressing one party's objection and drawing a fresh one from another (security, code-reviewer, and lucas42 in overlapping CHANGES_REQUESTED/APPROVE cycles). It was ultimately **closed unmerged**, and the fix shipped instead as a direct, unreviewed commit to `main` (`3ce4817`).

That churn had two concrete costs during the live incident:

- It consumed team attention on the code fix while the *production* outage (the credential itself) was still unresolved — the two ran in parallel and competed for the same responders.
- The direct-to-main shortcut that ended it **dropped the reviewed version's guard and its entire test suite**, shipping a live regression (`lucas42/lucos_creds#483`) into the exact code path this incident was about. The six rounds of review were bypassed at the last step, so the value that shipped had *less* review than any single reviewed round.

The process failure here is the coordinator's, and is named as such rather than sanitised: the round-count was itself the signal to force a convergence — a fix on its sixth round with each round trading one objection for another needs a decision, not a seventh relay. The coordinator relayed all six rounds instead of calling that, and has since amended its own instructions to treat repeated review rounds as a convergence trigger (`lucas42/lucos_claude_config@79e4db4`).

---

## What Was Tried That Didn't Work

- **First re-store + line-ending conversion (Round 2)** — intended to fix the CRLF corruption; instead introduced the trailing-newline-strip corruption. Full-length and structurally plausible, so it looked fixed and turned the dashboard green while sync stayed dead.
- **"Just redeploy to pick up the fix"** — moot at several points. A container only re-reads the credential on recreation, and re-pulling an unchanged (still-invalid) stored value changes nothing. Redeploy only helps *after* a valid key is in the store.
- **Measuring the key shape via `docker inspect --format` on the host** — an early responder reading of "36-char fragment" was a measurement artifact: `docker inspect` renders a multi-line env value across output lines, so a `grep '^VAR='` captures only the first line, making a full key look truncated. Corrected by measuring inside the container (`printenv | wc`, `ssh-keygen -l -f`), which is authoritative.
- **Trusting `/_info` / container health / the green dashboard** — the recurring trap this incident. The container passed every liveness signal in Round 2 while sync was completely broken. Only an actual sync run (or `ssh-keygen` on the deployed key) revealed the truth.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix the schedule-tracker false-recovery bug: `updateScheduleError`'s `INSERT OR REPLACE` nulls `last_success`, so a resumed-but-failing job flips green on its first error; preserve `last_success` and anchor staleness on the last *success* (shared `database.rb`, ~37 checks) | `lucas42/lucos_schedule_tracker#96` | Open (re-scoped; awaiting lucas42's confirm it's a bug) |
| Write-time validation in the credential store — name/identifier validation **plus a cheap type-agnostic reject of bare CR / control characters** (catches Round 1 at the write). Scoped *not* to include typed-value validation, per architectural review. | `lucas42/lucos_creds#473` | Open |
| Harden multi-line/key-typed secret handling end-to-end — base64 at rest + decode-then-validate at point of use (type-aware `ssh-keygen`/`openssl` dispatch; user-facing rejection message), retiring the divergent destination guards. Catches Round 2 and dissolves the transport class. Complementary to `#473`, not covered by it. | `lucas42/lucos_creds#484` (ADR-0006) | Proposed (PR awaiting lucas42 merge; implementation deferred to follow-up issues the ADR enumerates) |
| Fix the credential UI's CRLF-on-save corruption — the Round 1 write-path defect | `lucas42/lucos_creds#476` (commit `3ce4817`) | Done (closed `#476`; superseded PR `#477`) |
| Fix the regression in the shipped `#476` fix (`3ce4817`): undefined POST `value` throws `TypeError`; reinstate the reviewed test coverage that never landed on `main` | `lucas42/lucos_creds#483` | Open |
| Post-save credential readback (fingerprint/line-count) to catch a shape-valid-but-*wrong* value that no mechanical check can — the residual case beyond validation | `lucas42/lucos_creds#482` | Open (Awaiting Decision — lucas42 go/no-go; genuinely optional) |

---

## Sensitive Findings

[x] Yes — see note below.

A production SSH private key (`CONFIGY_SYNC_PRIVATE_SSH_KEY`) was central to this incident. No key material is included in this report: the key was only ever *measured* (byte length, line count, CR/LF counts, `ssh-keygen` validity, and public fingerprint) — never printed or reproduced. The public fingerprint of the good key (`SHA256:UAgm3fDnpms6X8OnZZqgH0MoWrXS8X876W3pZ7c2j74`) is non-sensitive by nature. Production credential *reads* are restricted to lucas42; the responder verified via the deployed value inside the container, which is the authoritative view of what the running service holds.
