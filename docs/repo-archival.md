# Repository Archival Process

A step-by-step checklist for retiring and archiving a lucos repository. Not every step applies to every repo — a static site has no volumes to clean up, and a script repo has no service to tear down. Use judgement, but work through the list in order.

## Phase 1: Pre-archival assessment

Before starting teardown, understand what you're removing and who might be affected.

- [ ] **Identify repo type.** Is this a deployed service (has a `domain` in configy's `systems.yaml`)? A script or library (`scripts.yaml`)? A component (`components.yaml`)?
- [ ] **Check for dependents.** Search the estate for references: `docker-compose.yml` files, `package.json` / `go.mod` imports, environment variables pointing to this service's domain, Loganne webhook config, and arachne ingestor `live_systems` (in `ingestor/triplestore.py`).
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

### 2b. Propagate to downstream systems

These systems derive their state from configy. After configy is updated:

- [ ] **Routing (lucos_router):** The router's daily cron job (22:16 UTC) regenerates nginx configs from configy and removes stale domain configs. To propagate immediately, SSH to the router host and run `update-domains.sh`, or redeploy the router.
- [ ] **DNS (lucos_dns):** Zone files are auto-generated from configy. Redeploy lucos_dns to regenerate, or wait for its next scheduled regeneration.
- [ ] **Monitoring (lucos_monitoring):** Service discovery happens at build time. Redeploy monitoring so it stops polling the retired service. Until redeployed, monitoring will alert on the now-unreachable service — if the teardown will take time, use the `PUT /suppress/{system}` endpoint to suppress alerts during the transition.
- [ ] **TLS certificates:** The router manages Let's Encrypt certs per domain. Removing the domain from routing means the cert will simply not be renewed and will expire naturally. No manual cleanup needed.

### 2c. Stop the service

- [ ] **Stop and remove containers** on the deployment host(s): `docker compose down` in the service's directory.
- [ ] **Remove Docker volumes** if the data is no longer needed. If the data might be needed for reference, take a final backup first. Remember: `docker compose down -v` removes volumes, but a plain `docker compose down` does not.
- [ ] **Remove the service directory** from the deployment host if it was cloned there.

### 2d. Clean up credentials

- [ ] **Check lucos_creds** for credentials belonging to this service (both credentials it owns and linked credentials where it's a client). Remove them.
- [ ] **Check `CLIENT_KEYS`** on other services — if the retired service was a client of other services, its token will be in their `CLIENT_KEYS`. Removing the linked credential from lucos_creds handles this automatically.

### 2e. Clean up arachne knowledge graph

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

- [ ] **Verify all open issues are closed.** GitHub prevents creating or reopening issues on archived repos.
- [ ] **Verify the project board is clean.** Items referencing archived-repo issues can become stale.
- [ ] **Archive the repo on GitHub.** Settings → Danger Zone → Archive this repository.

## Phase 5: Verification

After archiving, confirm the estate is clean:

- [ ] **Monitoring:** No alerts for the retired service. If monitoring hasn't been redeployed yet, do so now.
- [ ] **Routing:** The domain returns the router's default error page (or DNS no longer resolves, depending on timing).
- [ ] **Backups health check (`/_info`):** No "missing volume" warnings for volumes belonging to the retired service.
- [ ] **lucos_repos:** Next audit sweep completes without errors related to the archived repo.
- [ ] **configy validation tests:** Run configy's test suite to confirm no dangling references.

## Notes

- **Ordering matters.** Remove from configy before stopping the service. This ensures monitoring and routing stop expecting the service before it disappears, minimising false alerts.
- **Monitoring is the most latency-sensitive system.** It discovers services at build time, so it will keep polling a dead service until redeployed. Plan for this — either redeploy monitoring early, or suppress alerts.
- **Data retention.** If the service stored user data or data that took significant effort to create (check `recreate_effort` in configy's `volumes.yaml`), take a final backup before removing volumes. Err on the side of keeping backups — storage is cheap, regret is expensive.
- **Partial archival.** If a service is being replaced rather than retired, the successor service should be deployed and verified before the old one is torn down. Run both in parallel during the transition.
