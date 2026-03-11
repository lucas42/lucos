# lucOS Engineering Patterns

A reference for how lucOS systems are actually built and operated. This document describes the conventions, patterns, and standards in use across the ecosystem -- not aspirations, but reality as it stands today.

Last updated: 2026-03-11

---

## Repository structure

### One service per repo

Each deployable service lives in its own Git repository, hosted under `lucas42/` on GitHub. The naming convention is `lucos_{subsystem}` or `lucos_{subsystem}_{qualifier}` (e.g. `lucos_photos`, `lucos_photos_android`, `lucos_media_metadata_api`).

Multi-container services (e.g. an API + worker + database) are defined in a single repo's `docker-compose.yml`. The containers share the same repo, but each gets its own subdirectory and Dockerfile when they are custom-built images.

### Shared components

A small number of repos produce reusable artefacts rather than deployable services:

| Repo | Type | Consumed by |
|---|---|---|
| `lucos_navbar` | Web Component (npm package + Docker image) | ~20 services with a web UI |
| `lucos_search_component` | Web Component (npm package) | Services needing search UI |
| `lucos_deploy_orb` | CircleCI orb | All services via CI |
| `lucos_loganne_pythonclient` | Python package | Python services that emit Loganne events |
| `lucos_schedule_tracker_pythonclient` | Python package | Python services reporting to schedule tracker |

### Standard repo files

Most repos contain:

- `docker-compose.yml` -- always present for deployable services
- `.circleci/config.yml` -- CI/CD pipeline
- `.github/dependabot.yml` -- dependency update automation
- `.github/workflows/codeql-analysis.yml` -- static analysis
- `.github/workflows/auto-merge.yml` -- Dependabot auto-merge
- `CLAUDE.md` -- instructions for AI agents working on the repo
- `README.md`

---

## Languages and frameworks

The ecosystem is polyglot. Historically, this was a deliberate learning strategy -- each new service was an opportunity to pick up a new language, often starting with a from-scratch HTTP server before adopting frameworks. This explains the diversity: the language choices reflect what was being learned at the time, not a technical evaluation of which language was best for the job.

For new services today, there is no mandated language, but the practical considerations have shifted. With AI-driven development, the "learning a language" rationale is less relevant. When choosing a language for a new service, consider what makes sense for the problem and what the ecosystem already supports well.

### Languages currently in use

| Language | Services | Notes |
|---|---|---|
| **Node.js** | lucos_time, lucos_loganne, lucos_media_seinn, lucos_media_linuxplayer, lucos_notes, lucos_scenes, arachne explore container | Most use raw `http.createServer` or Express |
| **Python** | lucos_photos, lucos_contacts, lucos_eolas, arachne ingestor + MCP | FastAPI (photos), Django (contacts, eolas) |
| **Go** | lucos_media_metadata_api, lucos_configy (API), lucos_repos, lucos_creds, lucos_backups, lucos_schedule_tracker | Standard library or minimal dependencies |
| **Rust** | lucos_configy (API) | Single service |
| **Java** | lucos_media_manager | Maven, long-polling architecture |
| **PHP** | lucos_media_metadata_manager | Front-end for media metadata |
| **Erlang/OTP** | lucos_monitoring | In-memory state, email notifications |
| **Apache httpd** | lucos_root | Static site with config-time templating |

### Navbar consumption pattern

Services with a web UI include the `lucos_navbar` web component. The standard pattern uses Docker multi-stage builds:

```dockerfile
FROM lucas42/lucos_navbar:2.1.4 AS navbar
# ... later ...
COPY --from=navbar lucos_navbar.js ./static/
```

This pins a specific navbar version per service and avoids runtime npm installs.

---

## Docker and containers

### Container naming

Every container must have an explicit `container_name` in `docker-compose.yml`. The convention for new services is `lucos_{project}_{role}`, e.g. `lucos_photos_api`, `lucos_photos_postgres`.

Older services use shorter names. When the system was small, brevity made things easier to type and follow on the command line. As the ecosystem grew, knowing which system a container belongs to became more important than saving keystrokes, which is why newer containers always include the repo name as a prefix.

The intent is to migrate older names to the `lucos_` prefix over time, but renaming containers carries risk -- Docker may treat the rename as a new container rather than replacing the old one, leading to duplication rather than migration. This is why it has not been done in bulk.

| Pattern | Examples |
|---|---|
| `lucos_{project}_{role}` (current convention) | `lucos_photos_api`, `lucos_arachne_web`, `lucos_locations_mosquitto` |
| Short name (legacy, to be migrated) | `monitoring`, `root`, `time`, `loganne`, `seinn`, `authentication`, `notes` |
| Partial prefix (legacy, to be migrated) | `media_metadata_api`, `media_manager` |

### Image naming

Built containers should set `image:` following the pattern `lucas42/lucos_{project}_{role}`. This is used by the CI orb for pushing to Docker Hub.

### Environment variables

Environment variables in `docker-compose.yml` use **array syntax**, never `env_file`:

```yaml
environment:
  - SYSTEM                          # pass-through from .env
  - POSTGRES_PASSWORD               # pass-through from .env
  - POSTGRES_USER=photos            # hardcoded
  - REDIS_URL=redis://redis:6379    # hardcoded
```

Pass-through variables (no `=`) are sourced from the host environment or `.env` file. Hardcoded values that never vary between environments are set inline.

### Volumes

Named volumes must be declared in both the service's `volumes:` mount and the top-level `volumes:` section. Anonymous volumes are avoided because they lack Docker Compose project labels, which breaks `lucos_backups` monitoring.

Every named volume must also be registered in `lucos_configy/config/volumes.yaml` with a description and `recreate_effort` value (`automatic`, `small`, `tolerable`, `considerable`, `huge`, or `remote`).

---

## Networking and routing

### Architecture

All HTTP traffic is proxied through `lucos_router`, a shared Nginx reverse proxy. TLS termination happens at the router using Let's Encrypt certificates.

Each HTTP service is assigned:
- A **domain** (e.g. `photos.l42.eu`) -- configured in `lucos_configy/config/systems.yaml`
- An **HTTP port** -- the port the service listens on internally, also configured in `systems.yaml`

The router generates its Nginx config from `lucos_configy` and routes `{domain}` to `localhost:{http_port}`.

### DNS

DNS is managed by `lucos_dns` (BIND). Zone files are generated by `lucos_dns_sync` from `lucos_configy` data.

### Inter-service communication

Services on the same Docker Compose network communicate via service name as hostname (e.g. `redis://redis:6379`). Services in different compose stacks communicate via their public domain or `localhost:{port}` when on the same host.

---

## Secrets and configuration

### lucos_creds

All secrets and environment-varying configuration are managed by `lucos_creds`. Each service has a `.env` file per environment (development, production) that is fetched via SCP:

```bash
scp -P 2202 "creds.l42.eu:${PWD##*/}/development/.env"  .
```

### What goes where

| Where | What | Examples |
|---|---|---|
| **Hardcoded in `docker-compose.yml`** | Non-sensitive values that never vary | Database names, internal URLs, Redis URLs |
| **lucos_creds (`.env`)** | Secrets and environment-varying values | API keys, passwords, `PORT`, `APP_ORIGIN` |
| **lucos_configy** | Infrastructure metadata | Domains, ports, host assignments, volume registry |

### Standard environment variables

Every service receives from lucos_creds:

| Variable | Description |
|---|---|
| `SYSTEM` | System name (e.g. `lucos_photos`) |
| `ENVIRONMENT` | `development` or `production` |
| `PORT` | HTTP port |
| `APP_ORIGIN` | Public-facing base URL |

---

## CI/CD

### CircleCI

All services use CircleCI with the `lucos/deploy` orb. The standard pipeline is:

1. **Build** (`lucos/build-amd64`) -- builds Docker images and pushes to Docker Hub
2. **Test** (if tests exist) -- runs in parallel with build
3. **Deploy** (`lucos/deploy-avalon`) -- deploys to production, only on `main`, requires build (and test if present) to pass

ARM services use `lucos/build-arm64` or `lucos/build-armv7l` and deploy to `xwing` or `salvare`.

The build step only has access to a dummy `PORT` environment variable -- no other env vars are available at build time. This means `docker-compose.yml` must not use variable interpolation for anything other than `PORT` in the `ports:` mapping.

### Deployment hosts

The goal is to run as much as possible on avalon -- it sits in a datacentre with reliable networking and has the most compute resources. The ARM hosts exist for specific physical-world reasons that cannot be served from a remote datacentre.

| Host | Architecture | Location | Role |
|---|---|---|---|
| **avalon** | amd64 | Datacentre | Primary server. Runs the vast majority of services. |
| **xwing** | arm64 (Raspberry Pi) | Home (next to NAS) | Media file operations requiring low-latency access to the NAS (serving files, acoustic fingerprinting). The NAS is mounted as an NFS volume. Also runs the router and DNS for the home network. |
| **salvare** | arm64 (Raspberry Pi) | Home (different room) | Media playback. Raspberry Pis in different rooms have speakers attached for playing music where it is physically needed. |

Services that interact with the physical home environment (media playback, media file access) run on the ARM hosts. Everything else -- including `lucos_scenes` (home automation orchestration) -- runs on avalon, because its current scope is limited to calling other lucos services over HTTP rather than controlling local hardware directly.

### GitHub automation

Most repos have:
- **Dependabot** for automated dependency updates
- **Dependabot auto-merge** workflow for merging minor/patch updates automatically
- **CodeQL** for static security analysis
- **Code reviewer auto-merge** workflow (being rolled out) -- enables auto-merge on PRs approved by `lucos-code-reviewer[bot]`

---

## The `/_info` endpoint

Every HTTP service must expose `/_info` with no authentication. It returns a JSON object consumed by `lucos_monitoring` (health tracking) and `lucos_root` (homepage).

The full specification is in [`docs/info-endpoint-spec.md`](info-endpoint-spec.md).

Required fields: `system`, `checks`, `metrics`. Recommended: `ci`, `title`. Frontend-only: `icon`, `show_on_homepage`, `network_only`, `start_url`.

Health checks in the `checks` object are evaluated at request time. Each check has `ok` (boolean), `techDetail` (string), and optionally `debug` (string with error details).

---

## Event logging (Loganne)

`lucos_loganne` is the central event logging service. Services emit events for significant changes (data mutations, deployments, configuration changes) by POSTing to the Loganne API.

Not all services use Loganne -- it is adopted by services that mutate data (lucos_photos, lucos_contacts, lucos_eolas, lucos_media_metadata_api, lucos_creds) but not by read-only or infrastructure services.

The `LOGANNE_ENDPOINT` environment variable provides the Loganne URL to services that use it.

---

## Authentication

### User-facing authentication

User-facing services authenticate via `lucos_authentication` at `https://auth.l42.eu`. The auth domain is hardcoded in application code, not passed as an environment variable.

### API-to-API authentication (linked credentials)

Machine-to-machine authentication uses API keys managed by `lucos_creds`. When one service needs to call another, an administrator creates a **linked credential** in lucos_creds, specifying the client system/environment and the server system/environment. lucos_creds generates a random 32-character alphanumeric key and stores it (encrypted with AES-GCM).

The key is then automatically delivered to both sides via their `.env` files, but under different variable names and formats:

#### The client side: `KEY_{SERVER_SYSTEM}`

The calling service receives the key as a simple environment variable named `KEY_` followed by the server system name in uppercase. For example, if `lucos_photos` (production) has a linked credential to `lucos_arachne` (production), the photos `.env` file will contain:

```
KEY_LUCOS_ARACHNE="abc123..."
```

The client sends this as a bearer token (or `key` scheme) in the `Authorization` header when calling the server:

```
Authorization: key abc123...
```

#### The server side: `CLIENT_KEYS`

The receiving service gets a single `CLIENT_KEYS` environment variable containing **all** linked credentials from every client authorised to call it. The format is semicolon-delimited entries, where each entry is:

```
{client_system}:{client_environment}={key}
```

For example, a server with three authorised clients would receive:

```
CLIENT_KEYS="lucos_media_manager:production=abc123;lucos_photos:production=def456;external_calendar:production=ghi789"
```

The server parses this to build a lookup table of valid keys. When a request arrives, it extracts the token from the `Authorization` header and checks whether it exists in the lookup table. Some implementations also extract the client system and environment from the matching entry, making it possible to identify which client made the request.

#### Reserved variable names

lucos_creds enforces that certain keys cannot be manually created as simple credentials:

- `CLIENT_KEYS` -- reserved for server-side linked credentials
- `SYSTEM` and `ENVIRONMENT` -- reserved as built-in credentials
- Any key beginning with `KEY_` -- reserved for client-side linked credentials

This prevents accidental collisions between manually-set credentials and the automatically-generated linked credential variables.

#### How services parse `CLIENT_KEYS`

There is no shared library for parsing `CLIENT_KEYS` -- each service implements its own parser. This is a consequence of the polyglot ecosystem: with services written in Python, Go, Node.js, Java, PHP, and Erlang, a single shared library would not reach most consumers. A shared library could make sense for languages with multiple services (e.g. Python or Go), but has not been needed so far given the simplicity of the format.

The parsing logic is straightforward but must handle the format correctly:

1. Split the string on `;` to get individual entries
2. Split each entry on `=` (first occurrence only) to separate the client identifier from the key
3. Optionally split the client identifier on `:` to separate system from environment

Some services (like lucos_photos) only care about the key values for validation and discard the client identity. Others (like lucos_contacts) use the client identity to create per-client user objects with specific permissions.

---

## Databases

There is no single database technology. Services choose based on their needs:

| Database | Services | Notes |
|---|---|---|
| **PostgreSQL** | lucos_photos, lucos_contacts, lucos_eolas | Runs as a container in the service's compose stack |
| **SQLite** | lucos_media_metadata_api, lucos_repos, lucos_creds, lucos_schedule_tracker | Embedded, file-backed |
| **Redis** | lucos_photos | Used as a job queue (via RQ), not as a primary store |
| **Apache Fuseki** | lucos_arachne | RDF triplestore |
| **Typesense** | lucos_arachne | Full-text search index |
| **In-memory (with disk persistence)** | lucos_loganne, lucos_media_manager | Primary state in memory, periodically persisted to disk |
| **In-memory (volatile)** | lucos_monitoring, lucos_time | State held in process memory; lost on restart |

The original design philosophy was to avoid stateful services as much as possible, which is why several services hold their primary data structures in memory. Some of these have since had disk persistence retrofitted (notably lucos_loganne and lucos_media_manager), but the in-memory data structures remain the primary working copy -- the disk persistence may not be immediately obvious from the code.

PostgreSQL services run the database as a container within the same compose stack. There is no shared database server.

---

## Knowledge graph (Arachne)

`lucos_arachne` federates data from multiple lucos services into a unified knowledge graph. It consists of:

- **Ingestor** (Python) -- receives data pushes from source services, writes to Fuseki and Typesense
- **Fuseki** -- RDF triplestore with two endpoints: `raw_arachne` (read-write) and `arachne` (read-only, OWL reasoning)
- **Typesense** -- full-text search, consumed by `lucos_search_component`
- **Explore** (Node.js) -- web UI for browsing the knowledge graph
- **MCP server** (Python) -- provides structured tool access for LLM agents

Source services push their data to arachne when it changes (typically triggered by Loganne events or on a schedule). Arachne is not a source of truth -- canonical data always lives in the originating service.

---

## Monitoring

`lucos_monitoring` (Erlang) polls the `/_info` endpoint of every registered service. It tracks health status and sends email notifications when checks fail.

The monitoring status is available at `https://monitoring.l42.eu/api/status` (no authentication required).

`lucos_schedule_tracker` separately monitors scheduled/periodic tasks, tracking whether they run on time.

---

## Shared UI conventions

### Navbar

The `lucos_navbar` web component provides consistent top-level navigation across all services with a web UI. It is loaded as a standalone JavaScript file (not bundled) and used as a custom HTML element:

```html
<lucos-navbar>Service Name</lucos-navbar>
```

### Search

The `lucos_search_component` web component provides search UI backed by arachne's Typesense index. Used for entity lookup (e.g. searching for contacts to tag in photos).

---

## Inter-service communication patterns

### User-Agent convention

All inter-system HTTP requests should set the `User-Agent` header to the value of the `SYSTEM` environment variable. This makes it possible to trace which service initiated a request in logs and monitoring. Documented in [ADR-0001](adr/0001-user-agent-strings-for-inter-system-http-requests.md).

### Data flow

Data generally flows in one direction through the ecosystem:

1. **Source services** (contacts, eolas, media_metadata_api, photos) own canonical data
2. **Loganne** receives event notifications about data changes
3. **Arachne** ingests data from source services into the knowledge graph
4. **Consumer services** read from arachne's search index or triplestore

Services do not share databases. Integration happens through HTTP APIs and event notifications.
