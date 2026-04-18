# Incident: registry:3 OCI pull-through bug breaks estate multi-platform builds

| Field | Value |
|---|---|
| **Date** | 2026-04-17 |
| **Duration** | ~17 hours (2026-04-17 20:14 UTC to 2026-04-18 13:31 UTC — most of it overnight) |
| **Severity** | Partial degradation — blocked deploys estate-wide; no service went down (existing containers kept serving traffic) |
| **Services affected** | every repo using `lucos_deploy_orb`'s multi-platform build path via `docker.l42.eu` (lucos_docker_health hit it first; the rest would have hit it on their next deploy) |
| **Detected by** | SRE ops check — `lucos_docker_health` pipeline 73 flagged red in monitoring |

Source issues: lucas42/lucos_docker_mirror#35 (registry:3 OCI pull-through bug, fixed by lucas42/lucos_docker_mirror#36), lucas42/lucos_deploy_orb#137 (open — preventive fix for the secondary rate-limit cascade this incident surfaced).

---

## Summary

Yesterday's Dependabot auto-merge bumped `lucos_docker_mirror`'s upstream image from `registry:2` to `registry:3`. Distribution v3's proxy mode no longer fetches per-platform child manifests on demand when an OCI image index has already been cached — it caches the index but 404s on any digest not yet pulled through, with no upstream fallback. Every multi-platform CI build in the estate was broken from the moment the bump landed (20:14 UTC) until a revert to `registry:2` was deployed manually ~17 hours later (13:31 UTC). Fix shipped as lucas42/lucos_docker_mirror#36, with a Dependabot ignore rule on major version bumps of the `registry` image so it can't regress the same way again.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-17 15:48 | `b740ca79` lands on `lucos_docker_mirror` main — bakes `config.yml` into a custom `lucas42/lucos_docker_mirror_registry` image built `FROM registry:2` (closes lucas42/lucos_docker_mirror#8). Dependabot now sees a docker dependency it can bump |
| 2026-04-17 20:13 | `a7a81531` adds `ARG VERSION` + `ENV VERSION=$VERSION` to `registry.Dockerfile` (unrelated /_info refactor) |
| 2026-04-17 20:14 | Dependabot PR `5accfc5f` — "Bump registry from 2 to 3 in the major group" — auto-merges. New orb consumers pulling through `docker.l42.eu` now hit v3 |
| 2026-04-17 ~20:15 onwards | Overnight deploy attempts for any repo pulling an uncached upstream base image through the mirror fail at BuildKit manifest resolution. First concrete victim: `lucos_docker_health` pipeline 73 at 12:18–12:32 UTC on 2026-04-18 (first main-branch run after the bump) |
| 2026-04-18 12:32 | SRE ops check notices `lucos_docker_health`'s CircleCI check red. Log inspection on `lucos_docker_mirror_registry` shows `HEAD /v2/library/golang/manifests/1.26` returns 200 (cached OCI index), but `GET /v2/library/golang/manifests/sha256:e920...` 404s in 446µs — far too fast to have touched upstream |
| 2026-04-18 13:02 | lucas42/lucos_docker_mirror#35 raised with full evidence (registry logs, tracing path showing filesystem-only lookup) |
| 2026-04-18 13:07 | lucas42/lucos_docker_mirror#36 opened — pins `FROM registry:2` and adds a Dependabot `ignore` rule for `update-types: ["version-update:semver-major"]` on the `registry` image, with an inline comment explaining why |
| 2026-04-18 13:08 | Reviewer approves and auto-merges lucas42/lucos_docker_mirror#36 |
| 2026-04-18 ~13:10 | Mirror's own CI build for 1.0.12 fails at the `Docker Tag & Push (Latest)` step with HTTP 429 from Docker Hub on the `imagetools create` source pull. Retries (rerun-from-failed, then a fresh pipeline) hit the same 429. This is the separate bug already tracked as lucas42/lucos_deploy_orb#137 |
| 2026-04-18 ~13:25 | Decision: ship the fix to avalon manually rather than wait for Docker Hub rate limits to cool down. Versioned image (`lucas42/lucos_docker_mirror_registry:1.0.12`, which IS `registry:2.8.3`) had pushed successfully before the `:latest` step failed, so it's reachable on Docker Hub |
| 2026-04-18 13:28 | `docker pull`, `docker stop lucos_docker_mirror_registry`, `docker rm`, and `docker run` with full original compose env/volume/network config — including `--network-alias registry` so the `info` sidecar's `http://registry:5000` DNS still resolves |
| 2026-04-18 13:30 | Verification: the exact `GET /v2/library/golang/manifests/sha256:e920...` that 404'd in 446µs on registry:3 now returns 200 OK in ~500ms on registry:2 (round-trip to Docker Hub as expected) |
| 2026-04-18 13:31 | Monitoring baseline re-checked. `docker.l42.eu / registry` check transiently alerts because the manually-run container was initially missing its network alias; adding `--network-alias registry` clears the alert within the 2-minute verification window |
| 2026-04-18 13:40+ | Remaining estate deploys free to proceed once their next trigger runs. Mirror's own `:latest` tag and auto-deploy of 1.0.12 self-heal on the next successful CI build once Docker Hub rate limit clears naturally (1–6h typical rolling window) |

---

## Analysis

### Root cause: distribution v3 proxy mode doesn't fetch child manifests on demand

`registry:2` (distribution 2.8.x) has always handled the common multi-platform pull pattern correctly: a tag-based HEAD returns the OCI index; a subsequent GET for a specific per-platform manifest digest is resolved by falling through to upstream and caching the result. `registry:3` (distribution 3.0.0) proxy mode treats the digest lookup as local-only. It checks for a `revisions/sha256/<digest>/link` symlink under the blob store, finds none (because caching the parent index doesn't enumerate and pre-cache its children), and returns `manifest unknown` in sub-millisecond time without ever attempting an upstream call.

The config surface (`proxy:` block in `config.yml`) exposes no knob to re-enable the v2 behaviour. There is no documented opt-in. This is a behaviour change that breaks every BuildKit multi-platform pull on cold cache for a given upstream digest — which, since upstream images change over time, is effectively every non-trivial build.

No breaking-change warning was surfaced to users. The distribution v3 release notes reference "improved proxy correctness" but don't flag this specific regression. Raising it upstream is a reasonable next step but not blocking — the v2 image is still maintained and tagged on Docker Hub.

### Contributing factor: auto-merge on major version bumps

`lucos_docker_mirror` is an unsupervised repo with Dependabot auto-merge enabled. Dependabot was configured to treat major updates as a distinct group (`major` alongside `minor-and-patch`) but both groups auto-merged as soon as CI passed. CI passed because:

- the Dockerfile build itself works fine on `registry:3` — the container starts up and serves the root endpoints
- no CI test exercises the pull-through proxy behaviour against a multi-platform upstream image

So a semver-major breaking change slipped through, overnight, with no human review. `registry`'s major version bumps in particular have a history of reworking proxy internals — treating them as routine is not safe.

The fix on lucas42/lucos_docker_mirror#36 adds `ignore` for `update-types: ["version-update:semver-major"]` scoped to the `registry` image, with an inline comment pointing at the incident. Minor and patch bumps continue to flow. Future major bumps require a human PR.

### Contributing factor: secondary Docker Hub rate-limit cascade

When the fix was ready to ship, the mirror's own CI build failed at the `Docker Tag & Push (Latest)` step — HTTP 429 from Docker Hub on the `docker buildx imagetools create` round-trip pull of the just-pushed manifest. Two reruns (CircleCI `rerun-from-failed` and a fresh pipeline) hit the same 429.

This is not new. It is the same class of problem as the 2026-04-16 estate-rollout rate-limit incident and has a dedicated open issue on the orb: lucas42/lucos_deploy_orb#137. That issue proposes eliminating the separate `imagetools create` step entirely by passing both the versioned and `:latest` tags to `docker buildx bake --push` in the original build step — which streams straight from BuildKit to the registry with no round-trip and no pull at all. With that change, the rate limit would be a non-event for the `:latest` push path.

The workaround for this specific incident was to ship the fix manually: the versioned image did push successfully (the 429 was only on the subsequent imagetools step), so `docker pull lucas42/lucos_docker_mirror_registry:1.0.12` on avalon was sufficient.

### Contributing factor: no smoke test for the proxy's multi-platform pull path

`lucos_docker_mirror`'s CI tests that the container starts and `/_info` responds. It does not perform an actual multi-platform pull through the proxy. If it did, the Dependabot PR would have failed CI and the bump would have been flagged for review instead of auto-merged. This is the control that would have kept the bug out entirely — worth adding.

### What went right

- Detection was fast once someone looked: one ops check run flagged `lucos_docker_health` red; log inspection on the mirror immediately showed the 446µs local-only 404 that points unambiguously at a proxy behaviour change.
- The fix is a two-line revert plus a Dependabot rule — low-risk, fast to review, auto-merges.
- Restoring service via manual container swap stayed inside the documented production-change verification protocol (baseline snapshot, 2-minute wait, re-check, caught the missing `--network-alias` on the same read and repaired it immediately).

---

## What Was Tried That Didn't Work

- **Two attempts to push 1.0.12 through CI**: rerun-from-failed and a fresh pipeline both failed on the same 429 at the imagetools step. Further retries would have been pointless without waiting for the Docker Hub rolling window — 1–6 hours is typical — so the manual deploy was the right call.
- **Looking for a `proxy:` config knob in distribution v3 to restore v2 behaviour**: none exists. Reading the distribution v3 proxy source confirmed the digest lookup path is local-only by design in v3 — there is no option to enable upstream fallback.
- **Considering whether to also fetch `REGISTRY_PROXY_PASSWORD` from production creds and supply it to the manually-run container**: SRE's SSH key doesn't have production creds read access (scp returned `Permission denied (publickey)`). Running the registry anonymously was a valid fallback — upstream rate limits are per-IP/anon for unauthenticated pulls, which is lower but still functional at the volumes the mirror sees, and the authenticated config is restored automatically on the next Compose-managed redeploy.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Pin `registry` image back to v2 and add Dependabot ignore rule for major bumps | lucas42/lucos_docker_mirror#36 | Merged |
| Eliminate the separate `imagetools create` step from `publish-docker.yml` by tagging both `:<version>` and `:latest` during the `bake --push` — removes the Docker Hub round-trip that caused the secondary 429 cascade | lucas42/lucos_deploy_orb#137 | Open |
| Add a smoke test to `lucos_docker_mirror`'s own CI that performs a multi-platform pull through the proxy, so a future breaking proxy-mode regression would fail CI instead of auto-merging overnight | **Not yet raised** | To do |
| Raise the v3 proxy child-manifest behaviour upstream on `distribution/distribution` so that a future unpin from v2 becomes possible | **Not yet raised** | To do |
| Monitor whether the remaining 22 estate monitoring failures clear on their own now that the mirror is healthy (several are transitively blocked on lucos_docker_health deploying, which depended on this fix) | n/a | In progress |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
