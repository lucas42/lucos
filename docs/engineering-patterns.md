# lucOS Engineering Patterns

A reference for how lucOS systems are actually built and operated. This document describes the conventions, patterns, and standards in use across the ecosystem -- not aspirations, but reality as it stands today.

Last updated: 2026-06-07

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
| **Python** | lucos_photos, lucos_contacts, lucos_eolas, lucos_backups, arachne ingestor + MCP, dns sync container | FastAPI (photos), Django (contacts, eolas), raw `http.server` (backups) |
| **Go** | lucos_media_metadata_api, lucos_repos, lucos_creds | Standard library or minimal dependencies |
| **Ruby** | lucos_schedule_tracker | SQLite, minimal dependencies |
| **Rust** | lucos_configy | Single service |
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

### Scheduled jobs (supercronic)

Containers that run on a schedule use [supercronic](https://github.com/aptible/supercronic) — a container-native cron runner that propagates the container environment to jobs natively, logs stdout/stderr to Docker logs, and uses standard crontab syntax.

**Dockerfile pattern:**

Install supercronic with architecture-specific sha1sum verification, pinned to a specific release:

```dockerfile
ARG TARGETARCH
RUN set -e; \
    case "$TARGETARCH" in \
        amd64) sha1sum="5bcefed628e32adc08e32634db2d10e9230dbca0" ;; \
        arm64) sha1sum="639ab81a72771990790df7ee87d9acfe88e5fa83" ;; \
        *) echo "Unsupported architecture: $TARGETARCH" >&2; exit 1 ;; \
    esac; \
    wget -qO /usr/local/bin/supercronic \
        "https://github.com/aptible/supercronic/releases/download/v0.2.46/supercronic-linux-${TARGETARCH}"; \
    echo "${sha1sum}  /usr/local/bin/supercronic" | sha1sum -c -; \
    chmod +x /usr/local/bin/supercronic
```

Create a `crontab` file in the repo using standard cron syntax:

```
15 04 * * * /path/to/job.sh
```

Use supercronic as the container entrypoint:

```dockerfile
COPY crontab /crontab
CMD ["supercronic", "/crontab"]
```

**Containers with an additional long-running process** (e.g. an HTTP server alongside cron jobs) use a `startup.sh` that forks supercronic in the background and then starts the main process:

```bash
#!/bin/sh
set -e
supercronic /crontab &
exec pipenv run python -u server.py
```

**Upgrading supercronic:** Update the version string and both sha1sums together; fetch the sha1sums from the release assets at `https://github.com/aptible/supercronic/releases`.

---

## Networking and routing

### Architecture

All HTTP traffic is proxied through `lucos_router`, a shared Nginx reverse proxy. TLS termination happens at the router using Let's Encrypt certificates.

Each HTTP service is assigned:
- A **domain** (e.g. `photos.l42.eu`) -- configured in `lucos_configy/config/systems.yaml`
- An **HTTP port** -- the port the service listens on internally, also configured in `systems.yaml`

The router generates its Nginx config from `lucos_configy` and routes `{domain}` to `localhost:{http_port}`.

### DNS

DNS is managed by `lucos_dns`, which contains two containers: a BIND name server and a Python sync process that generates zone files from `lucos_configy` data.

### Real-time push: choosing a protocol

Three patterns exist in the estate for pushing updates from a service to its clients in real time. **SSE is the default** for one-way server→client push. Use **WebSocket** only when bidirectional traffic is genuinely required. **Long polling** is a recognised legacy pattern (currently `lucos_media_manager`); new code should not adopt it, and existing implementations may migrate to SSE when the relevant service is being touched anyway.

| Pattern | Direction | When to choose |
|---|---|---|
| **Server-Sent Events** | one-way (server → client) | Default for one-way push. Native browser API (`EventSource`), built-in auto-reconnect, plain HTTP semantics, no router cooperation needed. |
| **WebSocket** | bidirectional | The client genuinely needs to send messages back over the same connection. |
| **Long polling** | one-way (server → client) | Legacy only -- do not adopt for new code. |

### WebSockets

WebSocket connections are supported through `lucos_router` without any per-service configuration. The router's Nginx template includes a convention-based location block that matches two URL patterns:

- **`/stream`** -- exact match
- **Any path containing `/ws/`** -- regex match

For requests matching either pattern, the router adds the headers required for WebSocket upgrade (`Upgrade: $http_upgrade`, `Connection: "Upgrade"`) and switches to HTTP/1.1. All other proxy headers (host, forwarded-for, etc.) are passed through as normal.

This means a backend service only needs to listen for WebSocket connections on `/stream` (or a path containing `/ws/`), and the router will handle the upgrade transparently. No changes to `lucos_configy`, no bespoke Nginx config.

#### Backend pattern

Services that use WebSockets (currently `lucos_loganne` and `lucos_notes`) follow the same pattern. They use the `ws` npm package (`WebSocketServer`), attach it to the existing HTTP server, and bind to the `/stream` path:

```javascript
import { WebSocketServer } from 'ws';

export function startup(httpServer, app) {
    const server = new WebSocketServer({
        clientTracking: true,
        server: httpServer,   // share the HTTP server, not a separate port
        path: '/stream',
    });
    server.on('connection', async (client, request) => {
        // authenticate, then handle messages
    });
    app.websocket = {
        send: event => sendToAllClients(server, event),
    };
}
```

Key points:
- The WebSocket server shares the same HTTP server and port -- no additional port allocation needed
- Authentication uses the `auth_token` cookie from the upgrade request (parsed via `request.headers.cookie`)
- Unauthenticated clients are closed immediately with code `1008` (Policy Violation)

#### Client pattern

Browser clients connect using the page's own host and derive the protocol from the page URL:

```javascript
const protocol = location.protocol === "https:" ? "wss" : "ws";
const socket = new WebSocket(`${protocol}://${location.host}/stream`);
```

Clients implement automatic reconnection on close (typically with a short delay) since WebSocket connections are inherently less stable than HTTP requests.

### Server-Sent Events

Server-Sent Events (SSE) is the default for one-way server→client push. Browsers have a native `EventSource` API with built-in auto-reconnect, the wire protocol is plain HTTP/1.1, and `lucos_router` requires no special configuration to support it.

The convention path is **`/event-stream`** -- exact match. The path mirrors the SSE `Content-Type` (`text/event-stream`) verbatim and is a sibling-shape to `/stream` for WebSocket: both are "stream" patterns, with the protocol qualifier distinguishing them. If a service ever needs multiple SSE streams on one domain, use `/event-stream/<name>`.

#### Router-side: nothing to configure

Unlike WebSocket -- which requires `Upgrade`/`Connection: Upgrade` header pass-through and the explicit `/stream` carve-out in `lucos_router/templates/https.conf` -- SSE works through the router's default location block without any router cooperation. There is nothing to add to `lucos_router`, `lucos_configy`, or any per-service Nginx config.

#### Backend pattern

The handler is responsible for everything SSE-related:

- Set the SSE response headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`.
- Set **`X-Accel-Buffering: no`** -- nginx honours this header from the upstream specifically to disable response buffering for SSE. Without it, the router will buffer events and they will not reach the client in real time.
- Emit an SSE comment-frame heartbeat (`: ping\n\n`) every ~25s. nginx's default `proxy_read_timeout` is 60s, so a stream with no events will be closed by the router without a heartbeat. The browser's `EventSource` would then auto-reconnect, but the churn is avoidable and a heartbeat is cheap.

#### Client pattern

Browser clients connect to the same host as the page using the native `EventSource` API:

```javascript
const stream = new EventSource('/event-stream');
stream.addEventListener('message', (event) => {
    const data = JSON.parse(event.data);
    // ...
});
```

`EventSource` reconnects automatically on connection loss -- no client-side reconnection code needed.

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

### Naming network-resource variables: `*_ENDPOINT` vs `*_ORIGIN`

When introducing a new environment variable that refers to a network resource (another service, an event sink, an API), the suffix must reflect how the value is **consumed in code**:

| Suffix | Value contains | Used in code as |
|---|---|---|
| `*_ENDPOINT` | Full URL — origin **plus** path | Used as-is. Code does not append any path. |
| `*_ORIGIN` | Origin only — scheme + host + optional port, no path | Base URL. Code appends one or more paths. |

#### Examples in use

- `LOGANNE_ENDPOINT="http://172.17.0.1:8019/events"` — full URL with `/events` path baked in; consumers POST to this URL directly. **Conforms to `_ENDPOINT`.**
- `APP_ORIGIN="http://localhost:8015"` — base URL with no path; the consuming service builds multiple per-route paths from it. **Conforms to `_ORIGIN`.**
- `SCHEDULE_TRACKER_ENDPOINT="https://schedule-tracker.l42.eu/jobs"` — full URL including the `/jobs` path; the consumer (lucos_monitoring's `fetcher_scheduled_jobs`) reads from it as-is. **Conforms to `_ENDPOINT`.**

#### Picking the suffix

The decision is driven by code behaviour, not aesthetics:

- If the variable is read in exactly one place and that code does not do any string concatenation on it before calling `GET`/`POST`, the value is the endpoint — name it `*_ENDPOINT`.
- If the variable is read in multiple places, or the single read site appends a path (`f"{BASE}/foo"`, `urljoin(BASE, "/bar")`, etc.), the value is the origin — name it `*_ORIGIN`.
- If you find yourself wanting to call it `*_BASE_URL` or `*_HOST_URL`, you almost certainly mean `*_ORIGIN`.

#### Legacy `*_URL` (migration in progress)

The older suffix `*_URL` predates this convention and is being actively migrated (see lucas42/lucos#150). Remaining instances: `MEDIA_MANAGER_URL` (lucos_media_metadata_manager, lucos_media_linuxplayer, lucos_media_seinn) and `ARACHNE_URL` (lucos_media_metadata_manager).

- **New variables must use `*_ENDPOINT` or `*_ORIGIN`.** Pick one based on the consumption pattern above. Do not introduce new `*_URL` variables.
- When a service is being touched for other reasons, rename any `*_URL` it owns to the conforming suffix at the same time.

---

## CI/CD

### CircleCI

All services use CircleCI with the `lucos/deploy` orb. The standard pipeline is:

1. **Build** (`lucos/build-amd64`) -- builds Docker images and pushes to Docker Hub
2. **Test** (if tests exist) -- runs in parallel with build
3. **Deploy** (`lucos/deploy-avalon`) -- deploys to production, only on `main`, requires build (and test if present) to pass

Services that need to run on ARM hosts use `lucos/build-multiplatform` (docker buildx + QEMU emulation) instead of `lucos/build-amd64`, producing a multi-architecture image that works on both amd64 and arm64. These services deploy to `xwing` or `salvare`.

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

The client sends this in the `Authorization` header using the standard `Bearer` scheme:

```
Authorization: Bearer abc123...
```

> **Note:** Older code may use `Authorization: key abc123...` instead. Some services still accept the `key` scheme for backwards compatibility, but new code should always use `Bearer`.

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
