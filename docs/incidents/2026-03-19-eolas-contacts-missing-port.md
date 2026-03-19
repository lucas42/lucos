# Incident: eolas and contacts 502 — PORT missing from deploy .env

| Field | Value |
|---|---|
| **Date** | 2026-03-19 |
| **Duration** | ~1h46m (15:14 UTC to 17:00 UTC) |
| **Severity** | Partial degradation — two services down, cascading failures to two others |
| **Services affected** | eolas.l42.eu (502), contacts.l42.eu (502), auth.l42.eu (normalised-agentid check failed), schedule-tracker.l42.eu (googlesync_import errors) |
| **Detected by** | Monitoring alert — 9/31 systems showing errors |

Source issue: [lucas42/lucos#53](https://github.com/lucas42/lucos/issues/53)

---

## Summary

During a bulk CI push across ~30 repos, the production `.env` fetched from lucos_creds for `lucos_eolas` and `lucos_contacts` did not contain `PORT`. Docker Compose silently started both containers without host port bindings. Internal Docker healthchecks passed (they hit `127.0.0.1:80` inside the container), so the containers appeared healthy — but nginx could not reach them on their host ports. Both services returned 502 for ~1h46m. Retriggering CI for both repos restored the correct port bindings and resolved the incident.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 15:02 | lucos-system-administrator pushed bulk CI fix commits across ~30 repos (adding `CODE_REVIEWER_APP_ID`/`CODE_REVIEWER_PRIVATE_KEY` to GitHub Actions workflows), triggering simultaneous CI builds |
| 15:05 | `lucos/deploy-avalon` jobs started for lucos_eolas and lucos_contacts |
| 15:14–15:16 | `lucos_eolas_app` and `lucos_contacts_app` recreated without host port bindings (`docker port` returned empty); nginx begins returning 502 for both services |
| 15:22 | `lucos/deploy-avalon` job failed after 3 × 300s `docker compose up --wait` retries all timed out; `lucos_creds_configy_sync` logged `SSH Command timed out after 1 seconds` on `ls lucos_arachne/development/PORT` |
| 15:53 | configy_sync ran successfully with no changes logged, suggesting PORT was correctly set in creds by this point |
| 16:53 | configy_sync ran again successfully; PORT not written (already present) |
| 17:00 | SRE retriggered CI for eolas and contacts; both deploys succeeded |
| 17:00 | Port bindings restored; eolas.l42.eu and contacts.l42.eu return 200 |

---

## Analysis

### Stage 1: PORT absent from deployed .env

The `docker-compose.yml` for both services uses `$PORT:80` for the web container's host port mapping. When PORT is absent from the `.env` fetched at deploy time, Docker Compose silently starts the container without a host port binding — no error, no warning, exit code 0.

The exact reason PORT was absent from the fetched `.env` is unconfirmed. lucos_creds stores credentials in SQLite and serves `.env` files via SFTP. The database confirmed PORT was present throughout (`productionPORT` visible in `od -c` output). The SFTP transfer exited 0. lucos_creds container logs from before the 15:22 container restart are gone.

The best remaining hypothesis is a partial `.env` caused by a SQLite error under high concurrent SFTP load: `getNormalisedCredentialsBySystemEnvironment` calls four fetchers in sequence and returns early on any error. An error in one of the first fetchers would produce a valid but incomplete `.env` — missing credentials from the remaining fetchers — and `scp` would transfer the partial file with exit code 0 and no visible error. The simultaneous CI builds from the bulk push (~30 repos all fetching their `.env` around 15:05) are the circumstantial trigger.

### Stage 2: False healthy signal from Docker healthcheck

The internal Docker healthcheck for both services runs `wget http://127.0.0.1:80/_info` inside the container. This hits the port `gunicorn` is listening on internally, regardless of whether a host port binding exists. The containers reported `healthy` throughout. Docker, the deploy script, and external monitoring all saw a healthy container — nothing indicated the host port was missing until the router logs showed `connect() failed (111: Connection refused)`.

### Stage 3: Cascade to auth and schedule-tracker

`auth.l42.eu` depends on contacts for normalised-agentid resolution; with contacts returning 502, those checks failed. `schedule-tracker.l42.eu` runs `lucos_contacts_googlesync_import` — this also began erroring once contacts went down. Both services self-healed once contacts was restored.

---

## What Was Tried That Didn't Work

### Initial diagnosis: configy_sync deletion bug

The early hypothesis was that `configy_sync`'s `updateCredential` function was treating `None` as falsy and deleting PORT. This was incorrect — the behaviour is intentional (to propagate removals from configy to the creds database) and there were no `credentialDeleted` Loganne events to support it.

### `strings` search for PORT in SQLite

Running `strings /var/lib/creds_store/creds.sqlite | grep PORT` returned nothing, leading briefly to the incorrect conclusion that PORT was not in the database. In practice, SQLite stores the string `productionPORT` with no separator, so `strings` does not surface it as a standalone word. `od -c` confirmed PORT was present all along.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Make PORT a built-in credential in lucos_creds (not config-synced) | [lucas42/lucos_creds#109](https://github.com/lucas42/lucos_creds/issues/109) | Open |
| Add observability to lucos_creds for partial .env and SFTP errors | [lucas42/lucos_creds#110](https://github.com/lucas42/lucos_creds/issues/110) | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
