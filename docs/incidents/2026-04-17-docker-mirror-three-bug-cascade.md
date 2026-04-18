# Incident: Three-bug cascade in the Docker mirror stack breaks estate multi-platform builds

| Field | Value |
|---|---|
| **Date** | 2026-04-17 to 2026-04-18 |
| **Duration** | ~17h 17m from first breakage (2026-04-17 20:14 UTC) to last fix deployed (2026-04-18 13:31 UTC); acute phase 2026-04-18 ~11:44 UTC to ~13:31 UTC (~1h 47m) |
| **Severity** | Partial degradation — every multi-platform CI build in the estate blocked; no running service went down |
| **Services affected** | every repo using `lucos_deploy_orb`'s multi-platform build path via `docker.l42.eu`. First concrete victim: `lucos_docker_health`. Others would have hit the same failures on their next deploy |
| **Detected by** | SRE ops check — `lucos_docker_health` CircleCI check red in monitoring |

Source issues: lucas42/lucos_deploy_orb#132 (mirror probe misreads 401 as unavailable, closed by #133), lucas42/lucos_deploy_orb#134 (imagetools 404 via mirror on just-pushed digests, closed by #135), lucas42/lucos_docker_mirror#35 (registry:3 OCI pull-through bug, closed by lucas42/lucos_docker_mirror#36), lucas42/lucos_deploy_orb#137 (still open — preventive fix for the secondary rate-limit cascade surfaced during the third phase).

---

## Summary

Three distinct bugs in the `lucos_docker_mirror` stack broke every multi-platform CI build in the estate within a ~2-hour window. Each was unmasked by the previous fix:

1. **orb#132** — a mirror probe in `publish-docker.yml` treated the mirror as unavailable because it misread a correct 401 response. Builds fell back to direct Docker Hub pulls and hit 429 rate limits.
2. **orb#134** — once the probe was fixed, BuildKit routed `imagetools create` digest reads through the mirror, which 404'd on our own freshly-pushed digests.
3. **mirror#35** — with the orb using the mirror correctly, the multi-platform FROM pulls started failing too. Root cause: a Dependabot-merged `registry:2→3` bump the previous night had swapped in distribution v3, whose proxy mode no longer fetches child manifests on demand for cached OCI indexes.

All three fixes shipped on 2026-04-18. The final fix (mirror#35) also hit a secondary Docker Hub rate-limit cascade on the mirror's own CI build — tracked as the still-open orb#137. That cascade is unresolved but has a known fix path. Resolution was manual deployment of the registry:2 image to avalon via `docker pull` + `docker run`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-04-17 15:48 | `b740ca79` lands on `lucos_docker_mirror` main — bakes `config.yml` into a custom `lucas42/lucos_docker_mirror_registry` image built `FROM registry:2` (closes lucas42/lucos_docker_mirror#8). Dependabot now sees a docker dependency it can bump |
| 2026-04-17 20:14 | Dependabot PR `5accfc5f` — "Bump registry from 2 to 3 in the major group" — auto-merges. **Phase 3 trigger set.** No immediate breakage because no estate repo tries a multi-platform FROM pull through the mirror that night |
| 2026-04-18 11:44 | lucos_docker_health PR #73 merges; CircleCI pipeline 69 (build 299) starts on main |
| 2026-04-18 ~11:48 | **Phase 1 (orb#132) fires.** Mirror probe returns `curl: (22) … 401`, probe branches to "unavailable", BuildKit configured without mirror. The subsequent pull of `golang:1.26` direct from Docker Hub fails: `429 toomanyrequests: You have reached your pull rate limit as 'lucas42'` |
| 2026-04-18 ~11:50 | SRE investigation opens. lucas42/lucos_deploy_orb#132 raised with root cause (probe uses `grep '{'`, 401 body contains no `{`, probe misreads healthy 401 as failure) |
| 2026-04-18 11:58 | lucas42/lucos_deploy_orb#133 merged — probe rewritten to test reachability only |
| 2026-04-18 12:07 | lucos_docker_health build 319 (pipeline 71) re-triggered on the same commit. Mirror probe now succeeds, build+push succeeds, but `Docker Tag & Push (Latest)` fails at `docker buildx imagetools create` |
| 2026-04-18 ~12:10 | **Phase 2 (orb#134) fires.** Error: `content at https://docker.l42.eu/v2/lucas42/lucos_docker_health_app/manifests/sha256:c2a02a67… not found`. The image was just pushed directly to Docker Hub; BuildKit routes the digest read through the mirror (because the builder is configured with it); the mirror has never seen that digest; 404 |
| 2026-04-18 ~12:15 | lucas42/lucos_deploy_orb#134 raised — structural issue: global mirror config is right for upstream pulls, wrong for reading back our own just-pushed content |
| 2026-04-18 12:30 | lucas42/lucos_deploy_orb#135 merged — `imagetools create` now runs on `--builder default` (docker-driver, consults daemon config, no mirror) |
| 2026-04-18 ~12:32 | lucos_docker_health pipeline 73 re-triggered. Build+push succeeds, imagetools step now bypasses the mirror and succeeds, **but** the `build` step in the next run of multi-platform FROM pulls now fails at the mirror layer with `manifest unknown` |
| 2026-04-18 12:32 | **Phase 3 (mirror#35) fires.** Registry log inspection on avalon shows `HEAD /v2/library/golang/manifests/1.26` returns 200 (cached OCI index) but `GET /v2/library/golang/manifests/sha256:e920…` 404s in **446µs** — far too fast to have touched upstream. Root cause: distribution v3 proxy mode's local-only digest resolution, shipped by last night's Dependabot bump |
| 2026-04-18 13:02 | lucas42/lucos_docker_mirror#35 raised with full evidence (registry logs, tracing path showing filesystem-only lookup) |
| 2026-04-18 13:07 | lucas42/lucos_docker_mirror#36 opened — pins `FROM registry:2` and adds a Dependabot `ignore` rule for `update-types: ["version-update:semver-major"]` on the `registry` image, with an inline comment explaining why |
| 2026-04-18 13:08 | Reviewer approves and auto-merges lucas42/lucos_docker_mirror#36 |
| 2026-04-18 ~13:10 | Mirror's own CI build for 1.0.12 fails at the `Docker Tag & Push (Latest)` step with HTTP 429 from Docker Hub on the `imagetools create` source pull. Retries (rerun-from-failed, fresh pipeline) hit the same 429. This is the separate bug already tracked as lucas42/lucos_deploy_orb#137 |
| 2026-04-18 ~13:25 | Decision: ship the fix to avalon manually rather than wait for Docker Hub rate limits to cool. The versioned image (`lucas42/lucos_docker_mirror_registry:1.0.12`, which is `registry:2.8.3`) pushed successfully before the `:latest` step failed, so it's available on Docker Hub |
| 2026-04-18 13:28 | `docker pull`, `docker stop lucos_docker_mirror_registry`, `docker rm`, and `docker run` with full original compose env/volume/network config — including `--network-alias registry` so the `info` sidecar's `http://registry:5000` DNS still resolves |
| 2026-04-18 13:30 | Verification: the exact `GET /v2/library/golang/manifests/sha256:e920…` that 404'd in 446µs on registry:3 now returns 200 OK in ~500ms on registry:2 (round-trip to Docker Hub as expected) |
| 2026-04-18 13:31 | Monitoring baseline re-checked. `docker.l42.eu / registry` check transiently alerted because the manually-run container was initially missing its network alias; `--network-alias registry` added, alert cleared within the 2-minute verification window |
| 2026-04-18 13:31 onwards | Estate deploys free to proceed on next trigger. Mirror's own `:latest` tag and auto-deploy of 1.0.12 self-heal on the next successful CI build once Docker Hub rate limit clears naturally (1–6h typical rolling window) |

---

## Analysis

### Phase 1 — orb#132: probe misreads correct 401 as mirror-unavailable

#### Contributing factor: probe validated response body, not just reachability

The `Configure BuildKit registry mirror` step in `publish-docker.yml` was tightened in an earlier commit (`50b3932`, closing orb#122) to catch "degraded mirror states that would still fail under load" by validating the response body. The check looked for a `{` in the response to assert a JSON body. But `GET /v2/` on an authenticated registry correctly returns `401 Unauthorized` with a `WWW-Authenticate` challenge — no body content at all. curl with `-f` exits non-zero on 401; `|| true` prevents pipeline abort; `MIRROR_RESPONSE` ends up containing curl's error string (`curl: (22) The requested URL returned error: 401`), which contains no `{`. Probe concludes "unavailable", falls back to direct Docker Hub pulls.

The body-content check was intended to improve robustness but ended up being stricter than the registry protocol itself. Fixed in orb#133 by reducing the probe to a reachability test (any HTTP response = up; only network/DNS/timeout = fallback).

#### Contributing factor: no test for the probe against a correctly-configured registry

The probe was added as a robustness improvement without a test that actually runs it against a real `docker.l42.eu` (or equivalent) response. The Docker registry v2 protocol mandates 401 on unauthenticated `/v2/` when auth is configured; a probe that misreads this is a basic bug, but it shipped because no CI step exercised the probe itself against real registry behaviour.

### Phase 2 — orb#134: imagetools reads our own digests through the mirror

#### Contributing factor: global mirror config is wrong for registry reads from our namespace

Once the probe was fixed, BuildKit's custom builder was configured with `docker.l42.eu` as a mirror for `docker.io`. That mirror is appropriate for *upstream* pulls (base images like `golang:1.26`). It is *not* appropriate for the round-trip that `docker buildx imagetools create` performs when it reads per-platform manifest digests of an image that was pushed seconds earlier directly to Docker Hub. The mirror has never seen those digests; in proxy mode it looks them up locally, finds nothing, and 404s.

This is a design mismatch between "mirror all upstream pulls" and "mirror only when the target image isn't already in our own registry". The fix in orb#135 is to run `imagetools create` on the default docker-driver builder, which uses the daemon's registry config (no mirror) — simple, correct, and contained to this one step.

#### Contributing factor: the interaction with Phase 1 hid the bug

With Phase 1's broken probe, every build fell back to direct Docker Hub pulls, so `imagetools create` also went direct. Phase 2 was structurally latent the whole time the mirror config existed but was never hit because the probe always lost. Fixing Phase 1 immediately surfaced Phase 2 — a classic cascading-bug pattern where each fix reveals the next layer.

### Phase 3 — mirror#35: distribution v3 proxy mode doesn't fetch child manifests on demand

#### Root cause: behaviour change in distribution 3.0.0 proxy mode

`registry:2` (distribution 2.8.x) has handled the common multi-platform pull pattern correctly for years: tag-based HEAD returns the OCI index; subsequent GET for a per-platform child digest falls through to upstream and caches. `registry:3` (distribution 3.0.0) proxy mode treats digest lookup as local-only. It checks for a `revisions/sha256/<digest>/link` symlink, finds none (because caching the parent index doesn't enumerate and pre-cache its children), and returns `manifest unknown` in sub-millisecond time without ever attempting an upstream call.

The config surface (`proxy:` block) exposes no knob to restore v2 behaviour. No documented opt-in. The v3 release notes reference "improved proxy correctness" but don't flag this specific regression.

#### Contributing factor: auto-merge on semver-major Dependabot bumps

`lucos_docker_mirror` is an unsupervised repo with Dependabot auto-merge enabled. Major updates were placed in a separate `major` group but still auto-merged as soon as CI passed. CI passed because:

- the Dockerfile build itself works fine on `registry:3` — the container starts and serves root endpoints
- no CI test exercises the pull-through proxy against a multi-platform upstream image

A semver-major breaking change landed overnight with no human review. The registry image specifically has had major version bumps rework proxy internals in the past — treating them as routine auto-merge fodder is not safe. Fixed on the PR#36 side by ignoring `version-update:semver-major` for the `registry` image.

#### Contributing factor: secondary Docker Hub rate-limit cascade (orb#137)

When the fix was ready, the mirror's own CI failed at `Docker Tag & Push (Latest)` with HTTP 429 from Docker Hub on the `docker buildx imagetools create` pull. Two reruns hit the same 429. This is a third-order interaction — the same `imagetools create` round-trip that orb#134 moved off the mirror is now taxing Docker Hub for every estate deploy. Under rollout-scale concurrent load, the rate limit is close to the line; any single extra pull can tip into 429.

The fix direction is already defined in orb#137: pass both `:<version>` and `:latest` as tags to the original `bake --push` step, so BuildKit streams both tags at once from the build engine to Docker Hub with no pull round-trip. Not yet implemented, so the incident's final fix had to ship via manual `docker pull` + `docker run` on avalon. This worked cleanly because the versioned image did push successfully before the 429 hit the subsequent imagetools step.

### Across all three phases

#### Contributing factor: no end-to-end test of the mirror stack

The three phases share one deeper pattern: each bug would have been caught in CI by a test that actually exercised the affected path.

- Phase 1: a probe test that asserts "401 is healthy" would have caught orb#132 at PR time.
- Phase 2: a `publish-docker` end-to-end test that runs through `imagetools create` against a mirror-configured builder would have caught orb#134.
- Phase 3: a `lucos_docker_mirror` CI test that performs a multi-platform pull through the proxy would have caught the registry:3 behaviour change on the Dependabot PR.

None of these tests exist. The orb's CI exercises individual commands in isolation; `lucos_docker_mirror`'s CI tests that the container starts and `/_info` responds. Neither covers the combinations where these bugs live.

#### What went right

- Detection was fast and accurate. Each phase was identified from registry/BuildKit logs within minutes of firing.
- Each fix was small, contained, and reviewable in isolation. No one tried to fix everything at once.
- The production-change verification protocol caught the missing `--network-alias` on the manual container swap within the 2-minute monitoring window.
- The incident stayed within partial-degradation territory throughout — no running service went down. Existing containers kept serving traffic while deploys were blocked.

---

## What Was Tried That Didn't Work

- **Two attempts to push mirror 1.0.12 through CI**: rerun-from-failed and a fresh pipeline both failed on the same 429 at the imagetools step. Further retries would have been pointless without waiting for the Docker Hub rolling window (1–6h typical). Manual deploy was the right call.
- **Looking for a `proxy:` config knob in distribution v3 to restore v2 behaviour**: none exists. Reading the distribution v3 proxy source confirmed the digest lookup path is local-only by design in v3 — no option to enable upstream fallback.
- **Considering whether to fetch `REGISTRY_PROXY_PASSWORD` from production creds to supply to the manually-run container**: SRE's SSH key doesn't have production creds read access (scp returned `Permission denied (publickey)`). Running anonymously was a valid fallback — upstream rate limits are per-IP for anonymous pulls, which is lower but still functional at the volumes the mirror sees. The authenticated config is restored automatically on the next Compose-managed redeploy.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Mirror probe: reachability-only, stop misreading correct 401 as unavailable | lucas42/lucos_deploy_orb#133 | Merged |
| `imagetools create`: run on `--builder default` to bypass mirror | lucas42/lucos_deploy_orb#135 | Merged |
| Pin `registry` image back to v2 and add Dependabot ignore rule for major bumps | lucas42/lucos_docker_mirror#36 | Merged |
| Eliminate the separate `imagetools create` step from `publish-docker.yml` by tagging both `:<version>` and `:latest` during the `bake --push` — removes the Docker Hub round-trip that caused the secondary 429 cascade | lucas42/lucos_deploy_orb#137 | Open |
| Add a smoke test to `lucos_docker_mirror`'s own CI that performs a multi-platform pull through the proxy, so a future breaking proxy-mode regression would fail CI instead of auto-merging overnight | lucas42/lucos_docker_mirror#38 | Open |
| Add a `publish-docker` end-to-end test on the orb's own CI that exercises the mirror probe and `imagetools create` paths against a real registry response | lucas42/lucos_deploy_orb#138 | Open |
| Raise the v3 proxy child-manifest behaviour upstream on `distribution/distribution` so that a future unpin from v2 becomes possible | **Not yet raised** (upstream project — left for human decision) | To do |
| Monitor whether the residual estate monitoring failures clear on their own now that the mirror is healthy (several are transitively blocked on lucos_docker_health deploying, which depended on all three fixes) | n/a | In progress |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
