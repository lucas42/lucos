# ADR-0008: Deployment model has no on-host source of truth for compose-managed state

**Date:** 2026-05-28
**Status:** Accepted
**Discussion:** Surfaced by the 2026-05-28 xwing-network-flush incident (see `docs/incidents/2026-05-28-xwing-network-flush-orphaned-containers.md`); originally a tacit property of the deployment model rather than an articulated decision. This ADR is the retroactive promotion of that property to a named architectural commitment.

## Context

The lucos deployment model uses CircleCI as the deploy driver. For each service (per `lucos_deploy_orb/src/commands/deploy.yml`):

1. CircleCI runs `checkout` to bring the service's repository — including its `docker-compose.yml` — onto a fresh runner in `/home/circleci/project`.
2. The runner SCPs the production `.env` from `creds.l42.eu` onto the runner.
3. The runner runs `docker compose pull` and `docker compose up -d --wait` with `DOCKER_HOST=ssh://docker-deploy@<host>:<port>` — so the Compose CLI executes on the runner and reads the local compose file and `.env`, but the Docker daemon it instructs is on the remote production host.
4. When the workflow ends, the runner is destroyed. The `docker-compose.yml`, the `.env`, and any other deploy-time artefacts vanish with it.

What survives on the host is the **runtime state** of the Docker daemon: running containers, named volumes, user-defined networks, generated iptables rules. What does **not** survive on the host is any artefact from which that runtime state could be recreated. There is no checked-out compose file, no copy of `.env`, no rendered systemd unit, no host-side YAML, no shell script that recreates the network. The Docker daemon's own metadata (`/var/lib/docker/`) is the only record of the configuration; it is **derived** state, not source state.

This property has always been true. It was not previously documented anywhere. The 2026-05-28 xwing incident hit it head-on: a recovery procedure that flushed `/var/lib/docker/network/files/` to clear a stale IPv6 IPAM brought the daemon up with zero networks. The user-defined networks could not be recreated locally because there was no host-side compose file to recreate them from. Recovery required redeploying each affected service through CircleCI. The cost of the property became immediately visible; the property itself stayed correct.

## Decision

**On a lucos production host, the Docker daemon's runtime state is the only state. There is no on-host source of truth for compose-managed services, networks, volumes, or environment.**

The corollary, which governs all recovery procedures touching local Docker state:

> **Recovery from local Docker-state corruption is a CircleCI redeploy of every affected service, not an on-host `docker network create` / `docker volume create` / `docker compose up`.**

This applies even when an operator with host SSH access *could* in principle run `docker network create` against the daemon directly. Doing so produces local state that diverges from the source of truth in CI, fails to mirror any compose-network configuration (subnets, IPv6 enablement, custom drivers), and creates a recovery artefact that won't survive the next deploy. The only legitimate recovery shape is "redeploy through CI".

## Specifics

### What's compose-managed (covered by this ADR)

- Service containers — pulled by their compose files, recreated by `docker compose up`.
- User-defined networks (`<service>_default` and explicit `networks:` entries).
- Named volumes declared in compose, including the labels that `lucos_backups` reads to decide what to back up.
- Environment variables sourced from `.env` and rendered onto the container at start.

### What's not compose-managed (NOT covered by this ADR)

These are configured at the host level outside the deploy pipeline; recovery for them is host-level and is genuinely on-host work:

- The Docker daemon's `/etc/docker/daemon.json` — IPv6 settings, log drivers, storage drivers. Changes here happen by lucas42 editing the file on the host directly (recent example: lucos#192).
- Host network configuration outside Docker (`/etc/network/interfaces`, DNS, the host firewall — see [ADR-0007](0007-estate-wide-default-deny-port-policy.md) for the firewall model, which lives in a container but applies host-level rules).
- SSH host keys and the `docker-deploy` user account.

The distinction matters: when something at the daemon-config level is broken, the recovery is to fix the file on the host (with lucas42's hand on the keyboard). When something at the compose-managed level is broken, the recovery is to redeploy through CI.

### Recovery procedure shape

For any incident that leaves compose-managed state on a host in a broken or wiped condition, the recovery is **a CircleCI redeploy per affected service**, in deployment-graph order. Concretely:

1. Identify every service whose containers are on the affected host (see `lucos_configy` for the `service → hosts` mapping).
2. Trigger a redeploy for each, either by re-running the most recent CircleCI workflow or by pushing an empty commit. The redeploy will re-pull images, re-create user-defined networks, re-create named volumes (using existing data if `/var/lib/docker/volumes/<name>` still exists, fresh otherwise), and re-render `.env` from creds.
3. Verify external reachability per service (NOT just `docker ps` status — see the healthcheck-depth pattern documented in the SRE persona).

There is no shortcut. Operators with host SSH access should not run `docker network create` to "save time" — the resulting network won't match the compose definition and will be torn down on the next deploy, possibly silently.

## Consequences

### Positive

- **CI is the single source of truth, by construction.** No drift between "what's on the host" and "what CI last deployed" — because there's nothing on the host to drift. The most recent CI deploy *is* the configuration.
- **Disaster recovery from a wiped host is identical to fresh provisioning.** Spin up a new host with SSH access for `docker-deploy`, register it in `lucos_configy`, and redeploy every service that lists it in `hosts:`. There is no on-host bootstrap state to restore, no missing config-file backup to mourn.
- **Hosts are functionally interchangeable.** A service can deploy to any host that's in its `hosts:` list, because the deploy is hermetic from the host's perspective. (Architecture differences — amd64 vs arm64 — are the only constraint, and those are explicit in the deploy job.)
- **A compromised host cannot reproduce its own configuration.** Compose files, env files, and inter-service URLs are not present on the host filesystem in any usable form between deploys. Limits the lateral-movement value of a host compromise.
- **Recovery procedures are simple to write and to audit.** "Redeploy through CI" is a one-line runbook entry. There is no on-host script to maintain that could fall out of sync with the CI deploy logic.

### Negative

- **Live-state recovery requires a CI round-trip.** This is the cost we hit on 2026-05-28. When the network plane is corrupted, the operator cannot run a five-second `docker network create` to fix it — they must wait for a redeploy per affected service. For a six-service host like xwing, recovery time is dominated by the per-service deploy time multiplied by the number of services.
- **Recovery requires CircleCI to be working.** If CircleCI is down at the same time as a host's Docker state is broken, recovery is blocked. Mitigation: CircleCI outages are independent of any lucos infrastructure failure, so the joint probability is low; and the on-host runtime state survives a CircleCI outage (live-restore on the daemon side), so as long as nothing actively breaks the daemon during the outage, services continue running.
- **Recovery requires `creds.l42.eu` to be working.** The deploy pipeline fetches `.env` from creds via SCP. If creds is the affected service (as it is in any creds-related incident), recovery has a chicken-and-egg problem. `lucos_creds` itself uses `LUCOS_DEPLOY_ENV_BASE64` as a pre-supplied envfile path that bypasses the SCP — this is the documented escape hatch; do not remove it.
- **No quick "reapply current compose" path.** If a daemon-config change wants to be combined with a "re-run compose against the existing checked-out state", that state doesn't exist locally. Workaround: a manual operator can `git clone` the service repo on the host and run `docker compose up` against it, but that creates an artefact this ADR is explicitly designed to forbid. The right path is a redeploy from CI.
- **Recovery procedures must be written, not improvised.** Without an on-host source of truth, an operator can't reason from "what's on disk" to "what should be running". They must consult `lucos_configy` for the service → host mapping and the per-service repo for the compose file. Runbooks must encode that consultation step; ad-hoc recovery is risky.

### Out of scope

- **Daemon configuration recovery** (`/etc/docker/daemon.json`, host firewall, network interfaces) — covered above as "not compose-managed". The recovery shape for those is host-level, not CI-redeploy.
- **Volume data recovery.** Named volume *data* (e.g. Postgres data, photo files) is backed up separately by `lucos_backups`. This ADR is about the *configuration* that names the volume, not its contents. Restoring data after a wipe is its own runbook (`lucos_backups`); restoring the volume's existence is a redeploy.
- **In-flight requests or session state during recovery.** Any in-progress requests when the network plane breaks are lost. This ADR doesn't model graceful failover.

## Alternatives considered

### Cache compose files on the host post-deploy

Have CI `scp` the rendered `docker-compose.yml` and `.env` to a known path on the host as a final deploy step. Recovery scripts could then read from that path to recreate networks/volumes locally.

**Rejected.**

- Introduces a second source of truth that can drift from CI. A future incident where the on-host cache is stale (because a deploy aborted mid-cache-write) would re-introduce the failure mode this ADR formalises.
- Doesn't help without caching `.env` too, which expands the attack surface: secrets in cleartext on every host, persisted indefinitely, with new lifecycle questions about rotation and cleanup.
- The "five-second recovery" win is small relative to the architectural cost — the failure modes this ADR governs are rare enough (one significant incident to date) that the cost of caching does not pay back.

### Hosts pull compose from a known repository themselves (declarative reconciler)

A Kubernetes-style model where each host runs an agent that periodically reconciles its running state against a declarative spec checked into source control. Recovery becomes "wait for next reconcile" or "kick the reconciler".

**Rejected.**

- Massive architectural shift. We do not have the operational maturity, the team size, or the scale to justify a reconciler-based deployment model. The closest equivalent we have is `lucos_repos`' convention-audit loop, which is much smaller in scope.
- Reconcilers introduce a different class of failure mode (silent drift between desired and actual state, fight-loops between the reconciler and a hand-applied change). Trading the "no on-host source of truth" failure mode for the "reconciler edge cases" failure mode is not obviously a win.

### Push compose state to a config server (Consul, etcd) that hosts read

Separate the compose source-of-truth from CI: hosts read their desired state from a config server.

**Rejected.**

- Adds a new infrastructure component with its own availability, backup, and consistency story.
- The model becomes "configy + config-server + compose + CI" — more moving parts than the current "CI is the source of truth, configy is consulted for routing/firewall" — and the new component is on the critical path for every deploy.
- Cost-benefit poor at our scale. Worth revisiting only if the estate grows substantially.

### Status quo (CI is the only source of truth)

Selected. The cost is real (CI round-trip during recovery) but bounded, infrequent, and acceptable.

## Cross-references

- **Recovery runbook for the 2026-05-28 xwing incident** (under development by SRE) is the procedural counterpart to this ADR. It cites this document as the architectural justification for "redeploy, not recreate".
- [ADR-0007: Estate-wide default-deny port policy](0007-estate-wide-default-deny-port-policy.md) — the firewall is deliberately compose-managed (a container service) precisely to align with the architectural property this ADR formalises: its rules are regenerated from `lucos_configy` on each deploy, not stored on the host.
- `docs/incidents/2026-05-28-xwing-network-flush-orphaned-containers.md` — the incident that surfaced this property and motivated the ADR.
- `lucos_deploy_orb/src/commands/deploy.yml` — the implementation of the deploy model; `checkout` + transient `/home/circleci/project` + `DOCKER_HOST` over SSH is the mechanism by which "no on-host source of truth" is enforced today.
- `lucos_configy` — the service-to-host mapping that operators must consult to identify "every service on the affected host" during recovery.
