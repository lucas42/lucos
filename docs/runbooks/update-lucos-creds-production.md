# Runbook: lucos_creds itself — updating its production credentials, and recovering when it is down

This runbook covers the two situations where `lucos_creds` cannot be treated like any other system:

1. **[Updating a production credential whose system is `lucos_creds`](#why-lucos_creds-is-a-special-case).** For any other system, the normal `ssh -p 2202 creds.l42.eu …` command is sufficient.
2. **[`creds.l42.eu` is down](#if-credsl42eu-is-down-nothing-else-in-the-estate-can-build-or-deploy)** (for example, avalon is unavailable). No other project in the estate can build or deploy until `lucos_creds` is back, so it is restored first.

---

## Why lucos_creds is a special case

`lucos_creds` uses a self-deploy mechanism to avoid a circular bootstrap dependency: the deploy cannot SCP its own `.env` from `creds.l42.eu` (because that would require the service to already be running), so instead the deploy reads `.env` from a CircleCI project environment variable called `LUCOS_DEPLOY_ENV_BASE64` — a base64-encoded snapshot of the production `.env` file.

This means **two separate stores must be kept in sync**:

| Store | How it is updated | Used by |
|---|---|---|
| lucos_creds storage (the live key/value store) | `ssh -p 2202 creds.l42.eu lucos_creds/production/KEY=value` | All running services that read from `creds.l42.eu` at runtime |
| `LUCOS_DEPLOY_ENV_BASE64` CircleCI env var | CircleCI project settings (see below) | The deploy pipeline, which writes `.env` from this snapshot on every deploy |

**If you update only the lucos_creds value, the credential change silently fails to take effect.** The deploy writes `.env` from `LUCOS_DEPLOY_ENV_BASE64`, not from the live store — so the running service always gets the snapshot value regardless of what you set in lucos_creds storage. The new credential value never propagates to the running containers. This is the failure mode that caused the 2026-05-09 incident.

### Security note: credential rotations during incidents

This dual-update requirement has a critical security dimension when **rotating a credential after a suspected compromise**.

**Scope:** this risk applies only to credentials in lucos_creds's *own* `.env`. Credentials lucos_creds stores on behalf of other services and delivers via the SCP path are **not** affected — those never go through `LUCOS_DEPLOY_ENV_BASE64`. The credentials in scope are:

- `UI_PRIVATE_SSH_KEY`
- `CONFIGY_SYNC_PRIVATE_SSH_KEY`
- `KEY_LUCOS_CREDS` (the master credential for the credential store itself)

> [!WARNING]
> **Rotating any credential present in `LUCOS_DEPLOY_ENV_BASE64` without also updating the CircleCI env var means the rotation silently fails to take effect.** Credential values are written to `.env` from the snapshot at deploy time; the running service never sees the new value, with no error or warning.

During incident response — exactly when the pressure to act fast is highest and steps are most likely to be missed — this is the step that matters most. Even under pressure, follow the full five-step procedure below.

---

## Step-by-step procedure

### 1. Update the credential in lucos_creds storage

```bash
ssh -p 2202 creds.l42.eu lucos_creds/production/KEY=new_value
```

Verify it was accepted — the command returns an empty response on success.

### 2. Fetch the current production .env from lucos_creds

```bash
scp -P 2202 "creds.l42.eu:lucos_creds/production/.env" /tmp/lucos_creds_production.env
cat /tmp/lucos_creds_production.env
```

Confirm the file contains the new value before proceeding.

### 3. Base64-encode the .env file

```bash
base64 -w 0 -i /tmp/lucos_creds_production.env
```

The `-w 0` flag disables line wrapping — the CircleCI env var must be a single unbroken base64 string. Copy the output.

### 4. Update LUCOS_DEPLOY_ENV_BASE64 in CircleCI

1. Go to the [lucos_creds project settings in CircleCI](https://app.circleci.com/settings/project/github/lucas42/lucos_creds/environment-variables)
2. Find `LUCOS_DEPLOY_ENV_BASE64` and click **Edit** (or delete and re-add if the UI only allows replacement)
3. Paste the base64 string from step 3 as the new value
4. Save

**Alternatively**, using the CircleCI API:

```bash
# Replace TOKEN with a CircleCI personal API token
curl -X DELETE \
  -H "Circle-Token: TOKEN" \
  "https://circleci.com/api/v2/project/github/lucas42/lucos_creds/envvar/LUCOS_DEPLOY_ENV_BASE64"

curl -X POST \
  -H "Circle-Token: TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"LUCOS_DEPLOY_ENV_BASE64\", \"value\": \"$(base64 -w 0 /tmp/lucos_creds_production.env)\"}" \
  "https://circleci.com/api/v2/project/github/lucas42/lucos_creds/envvar"
```

### 5. Trigger a deploy and verify

Trigger a deploy by pushing a commit to `main` on `lucas42/lucos_creds` (or re-run the last pipeline in CircleCI).

Once the pipeline completes:

1. **Check the deploy logs** — the "Write .env file" step should not show errors
2. **Check `/_info`** — `curl https://creds.l42.eu/_info` should return HTTP 200 and show the expected build SHA
3. **Verify the credential is live** — test whatever service depends on the updated credential

---

## If the credential fails to take effect after a deploy

The most likely cause is that `LUCOS_DEPLOY_ENV_BASE64` was not updated (or was updated with the wrong value). To diagnose:

1. Inspect the credential in the running containers. For **SSH key credentials** (`CONFIGY_SYNC_PRIVATE_SSH_KEY`, `UI_PRIVATE_SSH_KEY`), each container writes its key to a file at startup — check the file:
   ```bash
   ssh avalon.s.l42.eu "docker exec lucos_creds_configy_sync cat /root/.ssh/id_ed25519"
   ssh avalon.s.l42.eu "docker exec lucos_creds_ui cat /root/.ssh/id_ed25519"
   ```
   For **other credentials**, check the environment variable directly:
   ```bash
   ssh avalon.s.l42.eu "docker exec lucos_creds_configy_sync printenv VARNAME"
   ssh avalon.s.l42.eu "docker exec lucos_creds_ui printenv VARNAME"
   ```
2. If the credential shows the old value, the snapshot is stale. Re-do steps 2–5 of this runbook.
3. If the credential shows the expected new value but the dependent service still fails, the issue is elsewhere — check the dependent service's container logs.

---

## If creds.l42.eu is down: nothing else in the estate can build or deploy

`lucos_creds` runs on avalon. Every project's CircleCI pipeline fetches credentials from `creds.l42.eu` (via `lucas42/lucos_deploy_orb`), so while it is down:

- **Builds fail.** The orb's `build` job always runs `fetch-publish-creds`, which scps `lucos_deploy_orb/publish/.env` from `creds.l42.eu`. There is no bypass.
- **Deploys fail.** The orb's `deploy` command scps the project's `production/.env` from `creds.l42.eu`, unless the project has a `LUCOS_DEPLOY_ENV_BASE64` CircleCI variable. Only `lucos_creds` has one.

This is deliberate. Other projects do not get their own copy of their credentials (in a CircleCI variable, a context, or a standby endpoint), because several sources for the same secrets drift apart and make credential management harder. `lucos_creds` carries a bypass only to solve its own bootstrap problem (lucas42/lucos#299).

**The rule: if `lucos_creds` is down, fix it first, before deploying anything else.**

### 1. Deploy lucos_creds by rerunning a previous pipeline

A fresh `lucos_creds` build cannot run, because the build step needs `creds.l42.eu`. Its deploy can, because of its `LUCOS_DEPLOY_ENV_BASE64` bypass. So redeploy an **already-published image** by rerunning only the deploy job of an earlier, successful pipeline. Nothing is built, so the build's credential fetch never runs.

1. Find `lucos_creds`'s **most recent** pipeline on `main` where both the build and the deploy succeeded. Check this. A pipeline that failed is not a usable rerun target, even if it is more recent.
2. Rerun just its deploy job, using CircleCI's selective job-rerun API (this is what worked during the 2026-09-15 rebuild):

   ```bash
   # WORKFLOW_ID: the workflow of the pipeline chosen in step 1
   # DEPLOY_JOB_ID: that workflow's deploy job (e.g. lucos/deploy-avalon), from GET /workflow/$WORKFLOW_ID/job
   curl -X POST \
     -H "Circle-Token: TOKEN" \
     -H "Content-Type: application/json" \
     -d "{\"jobs\": [\"$DEPLOY_JOB_ID\"]}" \
     "https://circleci.com/api/v2/workflow/$WORKFLOW_ID/rerun"
   ```

   **What gets deployed:** the orb's deploy picks its version from the repo's newest `v*` git tag (`git fetch --tags`), not from the rerun pipeline's own commit. So a rerun deploys the **latest released image**, with the `docker-compose.yml` from the rerun pipeline's checkout. Choosing the most recent successful pipeline keeps those two in step. A rerun of an older pipeline can pair an old compose file with a newer image.
3. Expect the workflow to show as **failed** even when the deploy worked. While avalon's other services are down, the final "Send deploy log to loganne" step fails, because loganne is on avalon too. The "Fetch deploy infrastructure credentials" step warns and carries on, skipping monitoring suppression. Check the deploy itself, not the workflow's colour: the "Deploy using docker compose" step must pass, and `https://creds.l42.eu/_info` must answer. Once loganne is back, you can rerun the workflow from failed to get a green record. That redeploys `lucos_creds` again, which is harmless once its store has been restored (step 2).
4. Check that the `LUCOS_DEPLOY_ENV_BASE64` snapshot is current. The deploy writes `.env` from it, so a stale snapshot brings `lucos_creds` up with stale credentials of its own. See the first half of this runbook.

### 2. Restore lucos_creds's data before anything else deploys

If the host was rebuilt, `lucos_creds` first boots with an **empty store** and freshly generated keys, until its `lucos_creds_store` volume is restored from backup (see `lucas42/lucos_backups`'s `docs/restore-runbook.md`). During that window, any other project's deploy would get missing or wrong credentials. So:

- restore the store immediately after `lucos_creds`'s first boot;
- don't let any other deploy run until the restore is done and verified. Hold off merges, and don't rerun other pipelines.

### 3. Then everything else

Once `lucos_creds` is up with its real data, deploys work again for every project. Restore the rest in the usual priority order (e.g. `lucos_configy`, then `lucos_dns`, `lucos_router`, `lucos_firewall`, then the rest).

**Fresh builds may still fail while avalon is only partly back.** `lucos_docker_mirror` is also on avalon, and a known orb defect (lucas42/lucos_deploy_orb#188) makes a build hard-fail against an unreachable mirror instead of falling back to Docker Hub. Until the mirror is up (or that defect is fixed), use the same technique as step 1: rerun only the deploy job of a previous successful pipeline, which skips the build.

### Don't

- **Don't deploy another project before `lucos_creds` by giving it a temporary `LUCOS_DEPLOY_ENV_BASE64`.** This was done once during the 2026-09-15 avalon rebuild, with a placeholder non-secret envfile for `lucos_root`, removed straight afterwards, purely to test the pipeline (lucas42/lucos#296). It is not the recovery procedure. The service gets credentials that aren't its real ones, and it creates exactly the second source of credentials this design avoids.
- **Don't add bypasses to other projects** to make the next outage easier. See the reasoning above.

Reference: the 2026-09-14 avalon disk failure, where this was the restore order ([incident report](../incidents/2026-09-14-avalon-disk-failure.md), section "Rebuild: nothing in the estate can build or deploy while `creds.l42.eu` is down").

---

## Cleanup

Remove the temporary file when done:

```bash
rm /tmp/lucos_creds_production.env
```

---

## Related

- [lucos_creds README — Setting or updating a credential](https://github.com/lucas42/lucos_creds#setting-or-updating-a-credential)
- [lucos_creds#304](https://github.com/lucas42/lucos_creds/issues/304) — issue tracking the discoverability gap this runbook addresses
- [lucos_creds#152](https://github.com/lucas42/lucos_creds/issues/152) — original self-deploy mechanism implementation
- [lucos#299](https://github.com/lucas42/lucos/issues/299) — decision that, when `creds.l42.eu` is down, `lucos_creds` is restored first and no other project gets its own credential copy
