# Runbook: Updating a production credential in lucos_creds itself

> **⚠️ This runbook applies only when you are updating a credential whose system is `lucos_creds` and whose environment is `production`.**
>
> For all other systems, the normal `ssh -p 2202 creds.l42.eu …` command is sufficient. Stop here if you are updating credentials for any other system.

---

## Why lucos_creds is a special case

`lucos_creds` uses a self-deploy mechanism to avoid a circular bootstrap dependency: the deploy cannot SCP its own `.env` from `creds.l42.eu` (because that would require the service to already be running), so instead the deploy reads `.env` from a CircleCI project environment variable called `LUCOS_DEPLOY_ENV_BASE64` — a base64-encoded snapshot of the production `.env` file.

This means **two separate stores must be kept in sync**:

| Store | How it is updated | Used by |
|---|---|---|
| lucos_creds storage (the live key/value store) | `ssh -p 2202 creds.l42.eu lucos_creds/production/KEY=value` | All running services that read from `creds.l42.eu` at runtime |
| `LUCOS_DEPLOY_ENV_BASE64` CircleCI env var | CircleCI project settings (see below) | The deploy pipeline, which writes `.env` from this snapshot on every deploy |

**If you update only the lucos_creds value and trigger a redeploy, the CircleCI snapshot overwrites `.env` with the stale value.** The change appears to deploy successfully but is silently reverted. This is the failure mode that caused the 2026-05-09 incident.

### Security note: credential rotations during incidents

This dual-update requirement has a critical security dimension when **rotating a credential after a suspected compromise**.

**Scope:** this risk applies only to credentials in lucos_creds's *own* `.env`. Credentials lucos_creds stores on behalf of other services and delivers via the SCP path are **not** affected — those never go through `LUCOS_DEPLOY_ENV_BASE64`. The credentials in scope are:

- `UI_PRIVATE_SSH_KEY`
- `CONFIGY_SYNC_PRIVATE_SSH_KEY`
- `KEY_LUCOS_CREDS` (the master credential for the credential store itself)

> [!WARNING]
> **Rotating any credential present in `LUCOS_DEPLOY_ENV_BASE64` without also updating the CircleCI env var will silently undo the rotation on the next deploy.** The old (potentially compromised) credential comes back with no error or warning.

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
base64 -w 0 /tmp/lucos_creds_production.env
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

## If a deploy appears to succeed but the credential reverts

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
