# ADR-0002: Expose built image version to the running application

**Date:** 2026-04-15
**Status:** Accepted
**Discussion:** https://github.com/lucas42/lucos_deploy_orb/issues/75

## Context

Once semver tagging at build time lands (lucas42/lucos_deploy_orb#73), every Docker image produced by a lucos CI pipeline will carry a version tag. The image will know what version it is, but the **running process inside the container** will not. Without an explicit mechanism to surface that version at runtime, `/_info` cannot report it, and operators cannot trivially answer the question "what code is currently live?" from a running service.

This matters for a few practical reasons:

- When investigating an incident, the first question is usually "what's actually deployed right now?". Today that answer requires cross-referencing CI, the registry, and the deploy host.
- When deploys become partially-applied (e.g. due to a stalled rollout, a broken host, or a silent rollback), the only reliable on-the-box signal of what is actually running is whatever the running process itself reports.
- Monitoring can use the version field to detect stale deploys — containers that are running an image older than the latest tag.

The problem therefore reduces to: how does the version string reach the running application?

Two shapes of solution are both technically viable, and both have been seen in other infrastructures. The trade-off between them is not subjective — one of them preserves a property (the image is the source of truth for what it is) that the other discards, and that property is load-bearing for the reason we are doing this at all.

### Option A: bake the version into the image at build time

- The `Dockerfile` accepts `ARG VERSION` and sets `ENV VERSION=$VERSION` (or writes to a file such as `/VERSION`).
- CircleCI passes `--build-arg VERSION=$NEXT_VERSION` when building the image.
- The application reads the env var (or file) at startup and includes it in its `/_info` response.

The version is a property of the image itself. Wherever that image runs — production, a local rebuild, a `docker pull` on a developer machine — the reported version matches the image exactly.

### Option B: inject the version as a runtime environment variable

- `docker-compose.yml` sets `environment: VERSION=${VERSION:-unknown}`.
- The value is sourced from lucos_creds, a deploy-time template substitution, or a CircleCI-driven env var.
- The application reads `$VERSION` at startup and reports it in `/_info`.

The version is a property of the orchestration layer rather than the image. The image is unmodified; the string is provided by whatever system composes the container at runtime.

## Decision

**Adopt Option A.** The version string is baked into the image at build time via a Dockerfile `ARG VERSION` / `ENV VERSION` pair, passed by CircleCI from `$NEXT_VERSION`.

Applications read `VERSION` from the environment at startup and include it in the `version` field of their `/_info` response (per [`docs/info-endpoint-spec.md`](../info-endpoint-spec.md)).

### Why Option A over Option B

The entire point of surfacing the version in `/_info` is so that a running container can tell us, with confidence, what code it is running. That confidence requires the version to be a property of the thing we are trying to identify — the image — not a property of a *different* system (the orchestration layer) that is meant to be describing it.

Option B reintroduces the class of bug we are trying to prevent. If lucos_creds lags, a docker-compose template drifts, or a deploy pipeline substitutes the wrong value, `/_info` will report a version that does not match the code actually running. Every layer of indirection between the image and the reported version is a place where the report can silently lie. In Option A there is no such indirection: the image carries its own identity, and any running copy of that image reports the same thing. Re-orchestrate it, move it to a different host, pull it locally for debugging — the version follows.

The practical cost of Option A is a one-line Dockerfile change in each of the ~40 repos plus a small application-side read. The cost of Option B is that the property we most want — the reported version matches the running image — becomes a convention that any of several independent systems can break without visible failure. That trade-off is not close.

This also matches the Twelve-Factor separation of build and run: build-time configuration (what this image is) belongs in the build artefact; run-time configuration (where it is deployed, how many replicas, what secrets it has) belongs in the orchestration layer.

### Mechanism

In each service's Dockerfile:

```dockerfile
ARG VERSION=unknown
ENV VERSION=$VERSION
```

The default `unknown` handles local builds that don't pass `--build-arg VERSION=...` and avoids a fatal startup failure when the variable is missing. The CI pipeline in `lucos_deploy_orb` passes `--build-arg VERSION=$NEXT_VERSION` during the build step.

Applications read `os.environ["VERSION"]` (or the language equivalent) at startup and surface it in `/_info` under the existing `version` field.

### Enforcement

A `lucos_repos` convention will be added to verify that each service's `Dockerfile` declares `ARG VERSION` and `ENV VERSION=$VERSION` (or equivalent). Without the convention, a Dockerfile that forgets these lines will silently report a blank or `unknown` version — a quiet failure mode that is exactly the kind of drift we're trying to avoid.

Per the estate-rollout guidance in the architect persona, the convention change and the per-repo Dockerfile updates are to be treated as a single piece of work routed via `/estate-rollout`, not split into separate issues. A draft convention PR makes the dry-run diff visible; the affected repos are migrated; then the convention is marked ready and merged.

## Consequences

### Positive

- **`/_info` reliably reports the running image's version.** No indirection through orchestration, so no drift class.
- **The image is self-describing.** Wherever it runs, the reported version matches. Useful for local reproduction of production incidents.
- **Stale-deploy detection becomes trivial.** Monitoring can compare the `/_info` version against the latest tag in the registry and alert on divergence.
- **Separation of concerns is clean.** Build metadata in the build output; runtime configuration in the orchestration layer. No compound values constructed in docker-compose.
- **No coupling to lucos_creds.** `VERSION` is not a credential or environment-varying value — it has no business being in a creds store.

### Negative

- **Dockerfile change in every repo (~40).** One-line mechanical edit per repo; rollout is sizeable but shallow.
- **Per-service application plumbing.** Each service needs a small change to read `VERSION` and include it in `/_info`. Already required by either option.
- **Silent failure mode if the Dockerfile skips the ARG.** `/_info` reports `unknown` rather than erroring loudly. Mitigated by the `lucos_repos` convention.
- **Image rebuild required to change the version.** That is the correct behaviour — a "new version" should always mean "a new image" — but it precludes workflows like "retag without rebuilding". None of our pipelines rely on that.
- **Local dev builds report `unknown`.** Acceptable: the whole point is that only CI-built images carry a real version. If a developer wants a real version locally, they pass `--build-arg VERSION=...` themselves.

### Follow-up actions

- Implement the build-side `--build-arg VERSION=$NEXT_VERSION` plumbing in `lucos_deploy_orb` (tracked in lucas42/lucos_deploy_orb#75).
- Add the `lucos_repos` Dockerfile-version convention **together with** the estate-wide Dockerfile update as a single `/estate-rollout`-routed issue. Do not split.
- Estate-wide rollout: update each service's application startup code to surface `VERSION` in `/_info` (can be phased, one language/framework at a time).
- Consider a future monitoring check comparing `/_info` version against the latest registry tag, once coverage is wide enough for that to be useful.
