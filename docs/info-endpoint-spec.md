# `/_info` Endpoint Specification

Every lucos HTTP service must expose a `/_info` endpoint. It requires no authentication and returns a JSON object describing the service. Two systems consume this endpoint:

- **lucos_monitoring** reads `system`, `checks`, `metrics`, and `ci` to track service health.
- **lucos_root** reads `title`, `icon`, `show_on_homepage`, `network_only`, and `start_url` to build the homepage.

## Field tiers

Fields are divided into three tiers based on how broadly they apply and how important they are for ecosystem integration.

### Tier 1: Required

Every service exposing `/_info` must include these fields. Their absence indicates a broken or non-compliant response.

| Field | Type | Description |
|---|---|---|
| `system` | string | The system name. Read from the `SYSTEM` environment variable. Used by monitoring as the canonical identifier. |
| `checks` | object | Live health checks, evaluated at request time. Each key is a check name; the value is a [check object](#check-object). Must be present. May be `{}` if there are no meaningful checks to report. |
| `metrics` | object | Live metrics, evaluated at request time. Each key is a metric name; the value is a [metric object](#metric-object). Must be present. May be `{}` if there are no meaningful metrics to report. |

**Why `checks` and `metrics` are required even when empty:** monitoring currently defaults these to `{}` if absent, which means a service that omits the field silently appears healthy. Making the field required -- even as an empty object -- means its absence is a signal of a non-compliant response rather than a service with nothing to report.

### Tier 2: Recommended

Strongly encouraged for full ecosystem integration. Consumers must handle their absence gracefully.

| Field | Type | Default if absent | Description |
|---|---|---|---|
| `ci` | object | `{}` | CI metadata. Currently only `ci.circle` (string, CircleCI project slug e.g. `"gh/lucas42/lucos_example"`) is consumed. |
| `title` | string | Falls back to `system` | Human-readable display name for the service. |

### Tier 3: Frontend services only

These fields are relevant only for services that have a user-facing web UI. API-only services should omit them. Consumers must not require them.

| Field | Type | Default if absent | Description |
|---|---|---|---|
| `icon` | string | _(none)_ | Path to the service's icon (e.g. `"/icon"`). Used by lucos_root to display on the homepage. |
| `show_on_homepage` | bool | `false` | Whether lucos_root should include this service on the homepage. |
| `network_only` | bool | `true` | Whether the service requires a network connection to render its UI. Services that work offline (e.g. via service workers) should set this to `false`. |
| `start_url` | string | `"/"` | URL path to the service's UI entry point. |

## Object shapes

### Check object

```json
{
  "ok": true,
  "techDetail": "Human-readable description of what is checked"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `ok` | bool | Yes | Whether the check is currently passing. |
| `techDetail` | string | Yes | A human-readable explanation of what the check verifies. |
| `debug` | string | No | Diagnostic detail, typically included only when `ok` is `false`. |

### Metric object

```json
{
  "value": 42,
  "techDetail": "Human-readable description of the metric"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | number | Yes | The current metric value. |
| `techDetail` | string | Yes | A human-readable explanation of what the metric measures. |

## Full example

A frontend service with health checks and metrics:

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
  "title": "Example",
  "icon": "/icon",
  "show_on_homepage": true,
  "network_only": true,
  "start_url": "/"
}
```

A minimal API-only service:

```json
{
  "system": "lucos_example_api",
  "checks": {},
  "metrics": {},
  "ci": {
    "circle": "gh/lucas42/lucos_example_api"
  },
  "title": "Example API"
}
```

## History

This specification was agreed in [lucas42/lucos#35](https://github.com/lucas42/lucos/issues/35) based on a review of the two consumers (`lucos_monitoring` and `lucos_root`) and the fields currently provided by services across the ecosystem.
