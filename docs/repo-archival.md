# Repository Archival Process

A step-by-step checklist for retiring and archiving a lucos repository. Not every step applies to every repo — a static site has no volumes to clean up, and a script repo has no service to tear down. Use judgement, but work through the list in order.

## Phase 1: Pre-archival assessment

Before starting teardown, understand what you're removing and who might be affected.

- [ ] **Identify repo type.** Is this a deployed service (has a `domain` in configy's `systems.yaml`)? A script or library (`scripts.yaml`)? A component (`components.yaml`)?
- [ ] **Check for dependents.** Search the estate for references: `docker-compose.yml` files, `package.json` / `go.mod` imports, environment variables pointing to this service's domain, Loganne webhook config, and arachne ingestor `live_systems` (in `ingestor/triplestore.py`). **Record the list of consumer services** — those with a `CLIENT_KEYS` or other env-var-referenced link to the retiring service — noting each service's container name(s) and the Docker host it runs on. This list is reused in Phase 2e; reconstructing it from scratch during credential cleanup is error-prone, especially under time pressure.
- [ ] **Review open issues.** Close all open issues with a comment: "Closing — this repository is being archived." Transfer any issues that are still relevant to a successor repo if one exists.
- [ ] **Remove from project board.** Remove all items for this repo from the "lucOS Issue Prioritisation" project board.

## Phase 2: Service teardown

*Skip this phase entirely if the repo is not a deployed service (i.e. has no entry in configy's `systems.yaml`).*

### 2a. Remove from configy

Configy is the single source of truth that drives routing, monitoring, DNS, and backup discovery. Update it first so downstream systems stop expecting the service to exist.

- [ ] **Remove from `systems.yaml`.** Delete the service's entry (domain, port, hosts).
- [ ] **Remove from `volumes.yaml`** if the service has any registered volumes.
- [ ] **Remove from `scripts.yaml`** or `components.yaml` if listed there.
- [ ] **Commit, push, and deploy configy.** The configy API must be live with the updated config before proceeding.

### 2b. Suppress monitoring alerts

Once configy has been deployed without the retired service, `lucos_monitoring` will continue polling it until monitoring itself is redeployed in Phase 2c — monitoring rebuilds its polling list at deploy time, not from a live configy fetch. Suppress alerts in this window so the polling gap doesn't fire spurious `monitoringAlert` events.

- [ ] **Call `PUT https://monitoring.l42.eu/suppress/{system}`** with the Bearer token in `lucos_agent/development/KEY_LUCOS_MONITORING`:

  ```bash
  scp -P 2202 "creds.l42.eu:lucos_agent/development/.env" ~/sandboxes/lucos_agent/.env
  . ~/sandboxes/lucos_agent/.env
  curl -X PUT -H "Authorization: Bearer ${KEY_LUCOS_MONITORING}" \
    "https://monitoring.l42.eu/suppress/{system}"
  ```

  Expected response: `204 No Content`. A `404` means the system isn't in monitoring's polling list (e.g. it's been removed already) — harmless, proceed.

  No lift step needed. Monitoring's in-memory suppression state is wiped at its next deploy (Phase 2c), and the retired system is no longer in the polling list afterwards, so the suppress entry becomes moot.

  **Suppression has a 10-minute TTL** (hardcoded in `lucos_monitoring/src/monitoring_state_server.erl`, originally sized for deploy windows). If Phase 2c is likely to take longer than 10 minutes — e.g. monitoring needs a non-trivial code change or its own PR review before redeploy — re-run the `PUT` to reset the timer. There's no harm in re-running pre-emptively.

  **Race:** between configy's deploy and this suppress call landing, monitoring's next poll cycle may catch a failure and emit a single `monitoringAlert`. In practice this is rare (the gap is seconds if you proceed straight from 2a to 2b) and self-limiting (subsequent polls are suppressed); no mitigation.

### 2c. Propagate to downstream systems

These systems derive their state from configy. After configy is updated:

- [ ] **Routing (lucos_router):** The router's daily cron job (22:16 UTC) regenerates nginx configs from configy and removes stale domain configs. To propagate immediately, run `docker exec lucos_router update-domains.sh` on the router host (the script lives inside the container, not on the host filesystem), or redeploy the router.
- [ ] **DNS (lucos_dns):** Zone files are auto-generated from configy. Redeploy lucos_dns to regenerate, or wait for its next scheduled regeneration.
- [ ] **Monitoring (lucos_monitoring):** Service discovery happens at build time. Redeploy monitoring so it stops polling the retired service. Alert suppression for the redeploy gap is already handled by Phase 2b.
- [ ] **TLS certificates:** The router manages Let's Encrypt certs per domain. Removing the domain from routing means the cert will simply not be renewed and will expire naturally. No manual cleanup needed.

### 2d. Stop the service

Note: there are no persistent service directories on production hosts (compose files are deployed transiently during CI and are not present afterwards). Use `docker stop`/`docker rm`/`docker volume rm` with the container/volume names directly rather than `docker compose down`.

- [ ] **Stop and remove containers** on the deployment host(s): `docker stop <container_names> && docker rm <container_names>`.
- [ ] **Remove Docker volumes** if the data is no longer needed. If the data might be needed for reference, take a final backup first. Use `docker volume rm <volume_name>` for each volume registered in configy's `volumes.yaml`.
- [ ] **Remove Docker networks.** Each compose project creates a `{project}_default` network on first deploy. Since compose files are not present on production hosts post-deploy, `docker compose down` cannot be used to remove it — remove manually: `docker network rm <project>_default` on each deployment host. First verify no containers are still attached: `docker network inspect <network> --format '{{len .Containers}}'` should return `0`.
- [ ] **Remove the service directory** from the deployment host if it was cloned there (rarely present — see note above).

### 2e. Clean up credentials

- [ ] **Check lucos_creds** for credentials belonging to this service (both credentials it owns and linked credentials where it's a client). Remove all of them, including `PORT` and `APP_ORIGIN`. The configy sync only writes credentials for systems currently in configy — it has no cleanup logic for removed systems, so orphaned credentials are not auto-deleted. Simple credentials (type `config` or `simple`) are deleted with `ssh -p 2202 creds.l42.eu "{system}/{env}/{key}="` (empty value = delete). Linked credentials (keys beginning `KEY_`) need the `rm` form: `ssh -p 2202 creds.l42.eu "rm {client}/{env} => {server}/{env}"`.
- [ ] **Check `CLIENT_KEYS`** on other services — if the retired service was a client of other services, its token will be in their `CLIENT_KEYS`. Removing the linked credential from lucos_creds handles this automatically.
- [ ] **Restart consumer containers.** Removing a linked credential from lucos_creds purges the env var from future deploys, but does not restart running containers — they continue to hold the old env until redeployed. For each consumer service in the list recorded in Phase 1, trigger a redeploy via CI (preferred — healthcheck-verified) or `docker restart <container_name>` on the host as a fallback. Without this step, orphan credentials can persist in process memory for months until the next unrelated deploy.
- [ ] **Reconcile downstream pre-registered state.** Some consumers persist derived state in their own datastores — state that survives both the retired service's shutdown *and* the consumer's own restart. For each consumer, identify whether the decommissioned service caused any of the following to be registered, and either revoke/delete it explicitly or confirm the consumer's own reconciliation logic will purge it at next startup:

  - **Typesense key store** (`lucos_arachne_search`): API keys are Raft-backed and survive container restarts. The `entrypoint.sh` stale-key-revocation loop reconciles them at startup — verify it will run cleanly, or manually delete via the Typesense admin API. *Worked example: the [2026-05-21 arachne-search restart-loop incident](docs/incidents/2026-05-21-arachne-search-typo-restart-loop.md) was triggered by two latent bugs in this loop reaching orphan `lucos_comhra` keys that had sat unreconciled for months.*
  - **PostgreSQL role grants**: if the retired service had a dedicated DB role, revoke grants and drop the role from any shared databases.
  - **SSH `authorized_keys`** on backup hosts: if the retired service had a deploy key for aurora or another SSH target, remove the public key from the relevant `authorized_keys` file.
  - **GitHub deploy keys / App installations**: if the retired service had a per-repo deploy key or a GitHub App installation, revoke it via the GitHub API or repository settings.

  If explicit revocation is not possible and reconciliation cannot be confirmed, document the residual orphan state in a comment on this archival issue and open a follow-up ticket.

### 2f. Clean up arachne knowledge graph

*Only if the service is listed in `live_systems` in `lucos_arachne/ingestor/triplestore.py` (currently: lucos_eolas, lucos_contacts, lucos_media_metadata_api).*

- [ ] **Remove from `live_systems`** in `ingestor/triplestore.py`. This dict maps system names to their data export URLs — each entry becomes a named graph in Fuseki.
- [ ] **Commit, push, and redeploy arachne.** On the next ingest run, the `cleanup_triplestore()` function will automatically drop the orphaned named graph and its triples.
- [ ] **Verify the graph is gone.** After the next ingest cycle, confirm the retired service's graph no longer appears in the triplestore.

## Phase 3: Estate-wide cleanup

These steps apply to all repo types, not just deployed services.

- [ ] **Remove from configy** (`scripts.yaml` or `components.yaml`) if not already done in Phase 2.
- [ ] **CircleCI:** The project will stop receiving builds once the repo is archived (no pushes). No manual action needed unless you want to clean up the CircleCI project settings.
- [ ] **GitHub Actions workflows:** Archived repos don't run workflows. No cleanup needed. Estate-wide rollout scripts should already skip archived repos (they fetch the repo list from GitHub, which includes the `archived` flag).
- [ ] **Dependabot:** Will stop running once the repo is archived. No manual cleanup needed.
- [ ] **lucos_repos audit:** Already skips archived repos (implemented in #90). No action needed.

## Phase 4: Archive the repository

- [ ] **Verify all open issues are closed, including any archival tracking issue.** GitHub archives make repos fully read-only — posting comments on issues (not just creating or reopening them) also returns 403 after archiving. Post any final completion comments (e.g. the summary comment the sysadmin persona requires before reporting back to the coordinator) **before** calling the archive API.
- [ ] **Verify the project board is clean.** Items referencing archived-repo issues can become stale.
- [ ] **Archive the repo on GitHub.** Settings → Danger Zone → Archive this repository.

## Phase 5: Verification

After archiving, confirm the estate is clean:

- [ ] **Monitoring:** No alerts for the retired service. If monitoring hasn't been redeployed yet, do so now.
- [ ] **Routing:** The domain returns the router's default error page (or DNS no longer resolves, depending on timing).
- [ ] **Backups health check (`/_info`):** No "missing volume" warnings for volumes belonging to the retired service.
- [ ] **lucos_repos:** Next audit sweep completes without errors related to the archived repo.
- [ ] **configy validation tests:** Run configy's test suite to confirm no dangling references.

## Phase 6: Notify the team

Once Phase 5 confirms the estate is clean, broadcast the decommissioning so each teammate can sweep their own persona file, memories, and any docs they own for stale references to the retired system.

**No broadcast mechanism exists.** `SendMessage` has no `to: "*"` or `to: "broadcast"` fan-out — each teammate must be messaged individually. Don't try a wildcard recipient; it'll go to a phantom inbox no-one reads.

- [ ] **Self-scan first.** Before broadcasting, sweep your own persona file (`~/.claude/agents/<your-agent>.md`), your memory directory (`~/.claude/agent-memory/<your-agent>/`), and any workflow / reference / ops-check files you own. Remove stale references to the retired system; update items where the lesson is still valid but the example needs replacing. You're not exempt just because you're driving the decom.
- [ ] **Send a `SendMessage` to each `lucos-*` teammate** (i.e. each persona file in `~/.claude/agents/lucos-*.md`, excluding your own). At time of writing the list is:

  - `lucos-architect`
  - `lucos-code-reviewer`
  - `lucos-developer`
  - `lucos-security`
  - `lucos-site-reliability`
  - `lucos-system-administrator`
  - `lucos-ux`

  team-lead is implicit (they orchestrated the decom and will sweep their own state as part of that — no need to send them a notification). If new lucos-* personas have been added since this doc was last updated, treat `~/.claude/agents/lucos-*.md` as authoritative over the list above.

  Suggested message template:

  > `lucas42/{repo}` has been decommissioned. Archival issue: `{URL}`. Please scan your persona file, memory directory, and any docs you own for references to `{system}` / `{repo}` — remove stale items, update items where the lesson is still valid but the example needs replacing. No reply needed unless you find something that needs cross-agent coordination.

  No need to wait for acknowledgements — each teammate cleans up asynchronously.

## Notes

- **Ordering matters.** Remove from configy before stopping the service. This ensures monitoring and routing stop expecting the service before it disappears, minimising false alerts.
- **Monitoring is the most latency-sensitive system.** It discovers services at build time, so it will keep polling a dead service until redeployed. Phase 2b handles this by suppressing alerts immediately after configy is removed, before downstream propagation starts.
- **Data retention.** If the service stored user data or data that took significant effort to create (check `recreate_effort` in configy's `volumes.yaml`), take a final backup before removing volumes. Err on the side of keeping backups — storage is cheap, regret is expensive.
- **Partial archival.** If a service is being replaced rather than retired, the successor service should be deployed and verified before the old one is torn down. Run both in parallel during the transition.
