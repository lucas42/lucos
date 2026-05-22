# Incident: lucos_arachne_search restart-loop from latent stale-key delete typos

| Field | Value |
|---|---|
| **Date** | 2026-05-21 |
| **Duration** | ~23 minutes (2026-05-21 23:53:50 UTC, first container crash, to 2026-05-22 00:16:54 UTC, container started healthy on hotfix image) |
| **Severity** | Partial degradation (search subsystem down; rest of arachne running) |
| **Services affected** | `lucos_arachne` (search subsystem only — web, mcp, explore, ingestor, triplestore all healthy throughout) |
| **Detected by** | Monitoring alerts (`lucos_arachne/search: fetch failed`, `lucos_arachne/circleci: build-deploy failed`), `lucos_docker_health/avalon` after 5 consecutive minute-polls; surfaced to SRE by lucas42 |

Source issue: [lucas42/lucos_arachne#556](https://github.com/lucas42/lucos_arachne/issues/556)

---

## Summary

A deploy of [lucas42/lucos_arachne#555](https://github.com/lucas42/lucos_arachne/pull/555) (cross-system Person merging in search index) triggered a redeploy of the `lucos_arachne_search` container on avalon. The new container's `entrypoint.sh` reconciled its typesense API keys against `CLIENT_KEYS` and found two orphan keys (`lucos_comhra:production` and `lucos_comhra:development`, left over from comhra's earlier decommissioning — only the first was visible in the initial diagnosis logs because the script exited on the first orphan encountered). It attempted to delete the first orphan via a curl call that contained two pre-existing latent typos — both naming undefined variables — producing a `DELETE /keys/` request with no auth header. Typesense replied 404, curl exited 22, `set -e` killed the entrypoint script and the typesense daemon along with it. Docker restarted the container, which crashed the same way; loop. The hotfix at [lucas42/lucos_arachne#557](https://github.com/lucas42/lucos_arachne/pull/557) corrects the two variable names; first deploy after merge cleanly revoked both orphans and restored service.

PR #555 itself does not touch the buggy code path and is not the cause — it was simply the first deploy after the orphan key was created that exercised the latent bug.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| (some earlier date) | `lucos_comhra` decommissioned; its production API key removed from arachne's `CLIENT_KEYS` env var, but search container not restarted at that time — key persisted in typesense's data volume as a benign orphan |
| 23:48:03 | PR [lucas42/lucos_arachne#555](https://github.com/lucas42/lucos_arachne/pull/555) merged to main |
| 23:50:52 | CircleCI `lucos/deploy-avalon` job started |
| 23:52:55 | New `lucos_arachne_search` container created on avalon |
| 23:53:50 | First crash logged (`SIGINT` after `curl: (22) The requested URL returned error: 404`) — docker begins restart loop |
| 23:55:12 | CircleCI `deploy-avalon` job marked **failed** (healthcheck never went green) |
| ~23:57 | First `lucos_docker_health/avalon` failure logged (assuming 5-consecutive-minute-poll threshold counted back from 00:02:00) |
| 2026-05-22 00:02:00 | `lucos_docker_health/avalon` 5th consecutive minute-poll failure; check goes red with debug `"Unhealthy containers: lucos_arachne_search"` |
| ~00:11 | lucas42 reports estate breakage to SRE via team-lead |
| ~00:12 | SRE pulls container logs, identifies typo bugs in `search/entrypoint.sh` lines 84 and 87; confirms via `git blame` that bugs pre-date PR #555 (since 2025-09-21, commit `5cf03f1b`) |
| ~00:14 | Incident issue [lucas42/lucos_arachne#556](https://github.com/lucas42/lucos_arachne/issues/556) filed |
| ~00:15 | Hotfix PR [lucas42/lucos_arachne#557](https://github.com/lucas42/lucos_arachne/pull/557) opened |
| 00:10:53 | PR #557 merged (auto-merge after code-reviewer approval, `lucos_arachne` is unsupervised) |
| 2026-05-22 00:16:54 | New `lucos_arachne_search` container started on hotfix image; entrypoint runs cleanly to completion, revoking both stale keys (`lucos_comhra:production` AND `lucos_comhra:development` — a second orphan the SRE had not anticipated, also cleaned up by the same fix). Healthcheck goes green. |
| 2026-05-22 ~00:17 | `lucos_arachne/search` returns to `healthy`. CircleCI `lucos/deploy-avalon` job marked **success**. |
| 2026-05-22 ~00:18 | `lucos_docker_health/avalon` returns to `healthy` once consecutive-error counter clears. Incident resolved. |

The PR-merge timestamp predating the issue-file timestamp is real — auto-merge on the hotfix fired while the SRE was still drafting the incident issue body, because code-reviewer approval landed on PR #557 before the issue body was finalised. Order in the table is preserved as-recorded.

---

## Analysis

### Root cause — two latent variable-name typos in `search/entrypoint.sh`

The "Delete stale keys" loop in `lucos_arachne/search/entrypoint.sh` (lines 81–89 pre-hotfix) referenced two variables that did not exist anywhere in the script:

```bash
KEY_ID=$(echo "$EXISTING_SYSTEMS_JSON" | jq -r ...)
#               ^^^^^^^^^^^^^^^^^^^^^ — should be $EXISTING_KEYS_JSON
curl -X DELETE "${TYPESENSE_URL}/keys/${KEY_ID}" \
    -H "X-TYPESENSE-API-KEY: ${TYPESENSE_ADMIN_SYSTEM}" --silent --show-error --fail
#                            ^^^^^^^^^^^^^^^^^^^^^^^^ — should be ${TYPESENSE_ADMIN_KEY}
```

The actual variable populated earlier in the script is `EXISTING_KEYS_JSON` (assigned on line 39 from `curl "${TYPESENSE_URL}/keys"`), and the canonical admin-key env var (validated on line 5 and used by every other curl in the script) is `TYPESENSE_ADMIN_KEY`. Bash's default behaviour for undefined variables is silent expansion to the empty string, so the runtime DELETE call was:

```bash
curl -X DELETE "${TYPESENSE_URL}/keys/" -H "X-TYPESENSE-API-KEY: " --silent --show-error --fail
```

Typesense returns 404 for `DELETE /keys/` (no key ID supplied). `--fail` flips the 404 into curl exit code 22, `set -e` at the top of the script exits the entrypoint, and the typesense daemon — started as a background process and `wait`-ed on at the end — is killed by the process-group teardown. Docker sees the container exit with code 22 (matching curl's exit code) and restarts it under the `restart: always` policy. Each restart re-runs the entrypoint, hits the orphan key again, and crashes identically. Loop.

Both typos have been latent in the file since commit `5cf03f1b` (2025-09-21, "add search component using typesense"), i.e. ~9 months. The code path was never exercised until 2026-05-21 because no key was ever removed from `CLIENT_KEYS` before then — the loop ran zero times.

### Trigger — orphan key from earlier `lucos_comhra` decommissioning, first deploy after

The orphans were created when `lucos_comhra` was decommissioned and its `lucos_comhra:production` and `lucos_comhra:development` API keys were removed from arachne's `CLIENT_KEYS` env var in `lucos_creds`. The search container was not restarted at that time, so the corresponding keys in typesense's data volume were never reconciled. They sat there as benign orphans: not in `CLIENT_KEYS`, no longer used, but still present in typesense. Only one (`:production`) was named in the diagnosis logs because the entrypoint exited on the first one it tried to delete; the second (`:development`) was only revealed once the fix landed and the entrypoint ran to completion and revoked both.

Any deploy after that orphan was created would exercise the buggy code path. PR #555 was simply the first.

This is a class of bug that is hard to detect by reading the diff of the change that triggered it: PR #555 added new field migrations to `entrypoint.sh` but did not touch the stale-key loop. The breakage was in code lines #555 didn't modify and didn't appear different from the day before in version control terms.

### Contributing factor — `set -e` plus typesense-as-child-process

The entrypoint runs typesense as a backgrounded child (`/opt/typesense-server ... &`), reconciles keys/schemas in the foreground, then `wait`s on the typesense PID. With `set -e`, any failed curl in the reconciliation block kills the script — and the typesense daemon along with it, because typesense is its child. This is reasonable defence-in-depth (don't run with broken auth/schema state) but it does mean entrypoint failures cascade into restart loops rather than partial-init states.

No change proposed here — the alternative (mask reconciliation failures with `|| true`) would mean a typesense daemon running with stale schema or unauthenticated keys, which is worse than a restart loop.

### Contributing factor — no shell linting on this script

`shellcheck` would have flagged both `$EXISTING_SYSTEMS_JSON` and `$TYPESENSE_ADMIN_SYSTEM` as referenced-but-unassigned (rule [SC2154](https://www.shellcheck.net/wiki/SC2154)). There is no `shellcheck` job in the search service's CI today. This is a cheap, deterministic guardrail that would have caught this exact class of bug at PR time, nine months ago. See follow-up [lucas42/lucos_arachne#558](https://github.com/lucas42/lucos_arachne/issues/558).

### Detection — monitoring caught it within ~9 minutes; user-report came ~17 min later

The first `monitoring` signal was `lucos_arachne/circleci: build-deploy failed` (the deploy-avalon job marking itself failed at 23:55:12), followed by `lucos_arachne/search: fetch failed` (the search check on /_info failing because the container was down) and `lucos_docker_health/avalon` (the docker-host health cron picking up the unhealthy container after its 5-consecutive-minute-poll threshold at ~00:02:00). The SRE was not paged but lucas42 noticed the alerts and routed to SRE via team-lead at ~00:11. From there, diagnosis-to-hotfix-merge took ~10 minutes.

The detection chain worked. The signal that flagged this most usefully was the `lucos_docker_health/avalon` debug string explicitly naming `lucos_arachne_search` as the unhealthy container — that pointed the SRE at the right container before any logs were read.

---

## What Was Tried That Didn't Work

Nothing significant — diagnosis followed a single trail: monitoring → container state → container logs → entrypoint script → git blame → confirm pre-existing. The typo was visible on first read of the script. No wrong hypotheses pursued.

The only marginal time cost was the SCP attempt to read `lucos_arachne/production/.env` from lucos_creds to confirm what was in `CLIENT_KEYS` directly. SCP-read on the `production` environment is restricted to lucas42, so this returned `Connection closed`. The SRE moved on without it — the symptom (`Revoking stale key (lucos_comhra:production)` in logs) was already sufficient evidence that `lucos_comhra:production` was in `EXISTING_SYSTEMS` but not in `DESIRED_KEYS`.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Original incident issue (root cause + hotfix) | [lucas42/lucos_arachne#556](https://github.com/lucas42/lucos_arachne/issues/556) | Closed by [lucas42/lucos_arachne#557](https://github.com/lucas42/lucos_arachne/pull/557) |
| Add `shellcheck` CI job to lucos_arachne to catch this class of typo (referenced-but-undefined variables) at PR time | [lucas42/lucos_arachne#558](https://github.com/lucas42/lucos_arachne/issues/558) | Open |

An estate-wide `shellcheck` rollout (covering every lucos repo with non-trivial entrypoint shell) is a plausible second-stage follow-up but is intentionally not filed here — the arachne-only issue should be implemented and judged useful first. If it is, the estate sweep can be raised as a separate ticket at that point.

---

## Sensitive Findings

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.

The orphan key `lucos_comhra:production` was the *description* of a typesense API key (not its value). Key values are stored in `lucos_creds`, never in the report. The orphan key's expected disposition (deletion) was correct and would have happened nine months ago if the script had worked; deleting it now is housekeeping, not a credential exposure.
