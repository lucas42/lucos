# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) for all projects in this environment.

## Lucos Infrastructure Reference

A reference for AI agents working on any lucos project. Covers deployment infrastructure, CI/CD, and conventions.

---

## Environment Variables & lucos_creds

Secrets and environment-varying config are managed by a service called **lucos_creds**. To write the local development `.env` file, run:

```bash
scp -P 2202 "creds.l42.eu:${PWD##*/}/development/.env" .
```

This is aliased as `localcreds` in the user's shell, but that alias is not available to Claude — use the raw command above.

### Standard vars always provided by lucos_creds

Every project gets these automatically:

| Variable | Description |
|---|---|
| `SYSTEM` | The system name (e.g. `lucos_photos`) |
| `ENVIRONMENT` | `development` or `production` |
| `PORT` | The port this service is exposed on |
| `APP_ORIGIN` | The public-facing base URL |

### Variable naming conventions

- External event infrastructure: `LOGANNE_ENDPOINT` (not `LUCOS_LOGANNE_URL` or similar)
- Authentication domain: always hardcode `https://auth.l42.eu` — do not use an env var for this
- Contacts API: `LUCOS_CONTACTS_URL`

### What goes where

- **Hardcode in `docker-compose.yml`**: non-sensitive values that never vary between environments (internal service URLs, fixed usernames, database names)
- **lucos_creds (`.env`)**: sensitive values and anything that varies between dev and production

Avoid constructing compound values (e.g. `DATABASE_URL`) in docker-compose using variable interpolation — the CI build step only has access to a dummy `PORT` and will fail if other variables are referenced. Instead, construct them in application code at startup (e.g. SQLAlchemy's `URL.create()`).

---

## Docker & Docker Compose

### Container naming

- `container_name` must be set on every container
- Names must follow the pattern `lucos_<project>_<role>` (e.g. `lucos_photos_api`, `lucos_photos_postgres`)

### Image naming (built containers only)

- Set `image:` on any container built from a Dockerfile
- Pattern: `lucas42/lucos_<project>_<role>` (e.g. `lucas42/lucos_photos_api`)

### Environment variables in compose

**Do not use `env_file`** — it breaks the CI build step.

Instead, declare every environment variable explicitly in the `environment` section using **array syntax**. This makes it clear which containers use which variables, and allows pass-through vars (sourced from the host environment / `.env` file) without specifying their values:

```yaml
environment:
  - SYSTEM                          # pass-through from host env
  - POSTGRES_PASSWORD               # pass-through from host env
  - POSTGRES_USER=photos            # hardcoded value
  - REDIS_URL=redis://redis:6379    # hardcoded value
```

Dictionary syntax cannot express pass-through vars without a value, so always use array syntax in `environment`.

Note: Docker Compose still reads `.env` for **compose-level** variable substitution (e.g. `${PORT}` in `ports:`) even without `env_file` — this is a separate mechanism and is fine to use.

### Build context

When multiple services share code (e.g. a shared Python package), set the build context to the repo root and specify the Dockerfile path explicitly:

```yaml
build:
  context: .
  dockerfile: api/Dockerfile
```

The Dockerfile can then `COPY shared/ /shared/` from the repo root.

### Networking

- HTTP traffic is proxied through a shared Nginx reverse proxy; TLS is terminated externally
- Services are exposed on `${PORT}`, configured per-environment via lucos_creds
- Containers on the same Docker Compose network communicate via service name as hostname

---

## CircleCI

Standard `.circleci/config.yml` for a lucos project:

```yaml
version: 2.1
orbs:
  lucos: lucos/deploy@0
workflows:
  version: 2
  build-deploy:
    jobs:
      - lucos/build-amd64
      - lucos/deploy-avalon:
          serial-group: << pipeline.project.slug >>/deploy-avalon
          requires:
            - lucos/build-amd64
          filters:
            branches:
              only:
                - main
```

- The `lucos/build-amd64` job builds and pushes Docker images
- The `lucos/deploy-avalon` job deploys to the server, but only on `main`
- The CI build step has access to a dummy `PORT` only — no other env vars are available during build

---

## GitHub Config

### CodeQL (`.github/workflows/codeql-analysis.yml`)

Only include languages actually present in the repo. Remove any languages copied from another project that don't apply (e.g. `javascript` in a Python-only project).

### Dependabot (`.github/dependabot.yml`)

Specify the correct directories for each ecosystem — these must match where the actual files live, not convention from the source project:

- `pip`: one entry per `requirements.txt` / `pyproject.toml` location (e.g. `/api`, `/worker`, `/shared`)
- `docker`: one entry per `Dockerfile` location (e.g. `/api`, `/worker`)
- `github-actions`: always `directory: "/"`

Remove any `ignore` rules that were specific to the source project's framework (e.g. Django's `asgiref`).

### Dependabot auto-merge (`.github/workflows/auto-merge.yml`)

Standard file, no project-specific changes needed.

---

## The `/_info` Endpoint

Every lucos HTTP service must expose a `/_info` endpoint with no authentication. It is used by monitoring and a homepage app.

Response is JSON with the following fields:

| Field | Type | Description |
|---|---|---|
| `system` | string | The system name — read from `SYSTEM` env var |
| `checks` | object | Live health checks (evaluated at request time). Each key is a check name; value has `ok` (bool) and `techDetail` (string) |
| `metrics` | object | Live metrics (evaluated at request time). Each key is a metric name; value has `value` (number) and `techDetail` (string) |
| `ci` | object | CI info: `{"circle": "gh/lucas42/<repo_name>"}` |
| `icon` | string | Path to the service's icon endpoint |
| `network_only` | bool | `true` if the UI requires a network connection to render; `false` if it works offline (e.g. via service workers) |
| `title` | string | Human-readable service name |
| `show_on_homepage` | bool | Whether to show this service on the lucos homepage |
| `start_url` | string | URL path to the UI entry point |

`checks` and `metrics` can be empty objects if there is nothing meaningful to report yet.

Example:

```json
{
  "system": "lucos_example",
  "checks": {
    "db-reachable": {
      "ok": true,
      "techDetail": "Checks whether a connection to PostgreSQL can be established"
    }
  },
  "metrics": {
    "photo-count": {
      "value": 42318,
      "techDetail": "Total number of photos stored"
    }
  },
  "ci": {
    "circle": "gh/lucas42/lucos_example"
  },
  "icon": "/icon",
  "network_only": true,
  "title": "Example",
  "show_on_homepage": true,
  "start_url": "/"
}
```
