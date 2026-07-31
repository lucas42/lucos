# Incident: media-metadata.l42.eu 500s — PHP alpha base image dropped mbstring

| Field | Value |
|---|---|
| **Date** | 2026-07-31 |
| **Duration** | ~6h29m — broken 07:46 UTC, restored 14:15 UTC. First *observed* failure 13:44 UTC; that nothing was observed before then is a property of traffic, not of impact (see Detection) |
| **Severity** | Complete outage of all field-rendering pages (the site's primary function) |
| **Services affected** | `lucos_media_metadata_manager` (media-metadata.l42.eu) on avalon. `lucos_media_metadata_api` unaffected. |
| **Detected by** | User report (lucas42, via team-lead). **Not** by monitoring — the estate board reported 55/55 healthy throughout. |

---

## Summary

A Dependabot PR (lucas42/lucos_media_metadata_manager#384) bumped the container base image from `php:8.5.8-apache-trixie` to `php:8.6.0alpha2-apache-trixie` and auto-merged. The alpha image does **not** bundle the `mbstring` extension, which the 8.5.8 image does. `src/views/field.php` calls `mb_strlen()` on every rendered form field, so every page that renders a field — album, collection and track views, i.e. essentially the whole site — died with `Uncaught Error: Call to undefined function mb_strlen()` and returned a bare HTTP 500.

The container **built**, **deployed**, and **reported healthy** the entire time. `/_info` kept returning 200 because it only probes the downstream API and touches no view code, so nothing alerted. The failure was found roughly six hours later by a user clicking a link.

Resolved by reverting the base image to `php:8.5.8-apache-trixie` (lucas42/lucos_media_metadata_manager#386), which also added a CI guard rejecting alpha/beta/rc base image tags so the identical bump could not be re-proposed and auto-merged the following morning.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 07:03:37 | Dependabot opens the `minor-and-patch` docker group bump: `php` `8.5.8-apache-trixie` → `8.6.0alpha2-apache-trixie`, classified `version-update:semver-minor` |
| ~07:11 | PR lucas42/lucos_media_metadata_manager#384 auto-merges; pipeline 981 starts |
| 07:21:36 | Image `1.0.129` built successfully — nothing in CI exercises it |
| 07:46:07 | `1.0.129` deployed to avalon; Apache starts under PHP 8.6.0alpha2. **All field-rendering pages are now broken** |
| 07:46 → 13:44 | No requests to field-rendering pages, so no 500s are logged. Monitoring green; container healthy |
| 13:44:38 | First user-visible 500 — the first `mb_strlen()` fatal in the container log |
| ~13:47 | lucas42 reports 500s with no error message, and that nothing is alerting; SRE engaged via team-lead |
| 13:49 | Live probe: `/_info` returns 200 and healthy; root returns the normal 302 to aithne — the outage is invisible from the endpoints monitoring watches |
| 13:52 | Root cause identified from the container log; confirmed directly — `php -r 'var_dump(function_exists("mb_strlen"));'` → `bool(false)` in the running container, and `php -m` on the previous image `1.0.128` lists `mbstring` |
| 13:52:49 | Hotfix PR lucas42/lucos_media_metadata_manager#386 opened (revert base image + CI guard against pre-release tags) |
| 13:55:06 | PR approved by lucos-code-reviewer (~2 min turnaround) |
| 13:55 → 14:13 | PR waits on a second approval. The repo is supervised, so agent approval alone cannot merge — ~18 minutes of the resolution time is this wait |
| 14:13:01 | PR approved by lucas42 → auto-merged 14:13:14; deploy pipeline 985 starts |
| 14:15:18 | `1.0.130` deployed to avalon, container healthy — **service restored** |
| ~14:20 | Verified: `mbstring` present, `mb_strlen("composer_name")` → 13, and `views/field.php` renders end-to-end producing the expected `class="key-label long-key"` markup |

---

## Analysis

### Root cause: the alpha image doesn't ship mbstring

`php:8.6.0alpha2-apache-trixie` does not bundle `mbstring`. This was confirmed directly rather than inferred, by comparing the two images on the production host:

| Image | `php -m \| grep mbstring` |
|---|---|
| `lucos_media_metadata_manager:1.0.128` (php 8.5.8) | `mbstring` present |
| `lucos_media_metadata_manager:1.0.129` (php 8.6.0alpha2) | absent |

and by reproducing the exact failing call inside the running container:

```
$ docker exec lucos_media_metadata_manager php -r 'var_dump(function_exists("mb_strlen"));'
bool(false)
```

`src/views/field.php:12` uses `mb_strlen($key)` to pick a CSS class based on label length. Every form field renders through that view, so the fatal fired on every album, collection and track page. The stack traces in the container log confirm the actual failing requests took that path:

```
PHP Fatal error:  Uncaught Error: Call to undefined function mb_strlen()
  in /srv/metadata_manager/views/field.php:12
#0 /srv/metadata_manager/views/album.php(44): include()
#1 /srv/metadata_manager/controllers/viewalbum.php(45): require('...')
#2 /srv/metadata_manager/html/albums.php(98): viewAlbum()
```

Worth stating plainly: PHP 8.6.0alpha2 is a **pre-release**. Whether the unbundling is a deliberate 8.6 change or an artefact of the alpha build is not established here, and does not need to be — the defect is that a pre-release image reached production at all.

### Contributing factor: an alpha was classified as a routine minor bump and auto-merged

Dependabot reported the update as `version-update:semver-minor` and placed it in the `minor-and-patch` group, which auto-merges. Nothing in the estate distinguishes "a patch release of a stable line" from "an alpha of the next major-minor" — both look like routine hygiene, and the auto-merge workflow treats them identically.

### Contributing factor: CI has never tested the image we ship

The `test` job runs on `cimg/php:8.4`, an environment unrelated to the built image, which *does* bundle `mbstring`. So phpunit passed against a PHP that production doesn't run. A second, independent gap compounds it: no test in `tests/Unit/` renders a view, so even running the existing suite inside the production image would have gone green through this incident.

This is the same shape as the June mail outage (`2026-06-11-mail-alpine-dovecot24-outage.md`): the image builds, so the auto-merge gate passes, and the breakage only appears at runtime. In that case the container crash-looped and the deploy healthcheck failed, so CI went red and the problem announced itself within minutes. Here, the container started perfectly and served `/_info` happily — so the same class of defect produced a *quieter* and much longer outage.

### Contributing factor: detection had nothing to detect with

`/_info` declares one check, `metadata-api`, probing `GET /v3/tracks/1` on the downstream API. That dependency was healthy. The container healthcheck curls the same `/_info`. Neither touches view code, so the service reported itself perfectly well while serving 500s to every user.

This is `/_info` behaving as specified — availability and dependency config, not content-rendering correctness — but the practical consequence is that a total user-facing outage of this service was invisible. The six-hour gap was luck, not detection: the breakage started at 07:46 and was only noticed when someone happened to browse at 13:44. Detection latency wasn't slow; detection *capability* was zero.

### The trap: the obvious remedy would also have gone green

This is the most important paragraph in the report, and it was only implicit until lucos-architect flagged it.

The natural conclusion to draw from everything above is "add a CI job that boots the built image and curls `/_info`" — it's the shape used elsewhere in the estate, and it sounds like it directly addresses "the image was broken and nothing noticed". **It would have passed cleanly through this entire outage.**

`src/html/_info.php` requires `api.php` and nothing else. It never includes `views/field.php`, never renders a field, and never touches any code path that needs `mbstring`. A CI job booting `1.0.129` and curling `/_info` would have got a healthy 200 — exactly as production did for six and a half hours.

The same trap applies to the `FROM app AS test` pattern (lucas42/lucos_eolas), which is otherwise the strongest candidate for the durable fix: running tests inside the shipped image only helps if some test exercises the broken path, and nothing in `tests/Unit/` renders a view. Environment parity and coverage are **independent** requirements, and parity alone would have shipped this outage too.

The generalisable lesson is uncomfortable: a guard that boots the artefact and asks it whether it's well proves only that the artefact can answer questions. It has to be asked to do the thing it exists to do.

### Aggravating factor: the 500 page said nothing

PHP runs with `php.ini-production`, so `display_errors` is off — correct for production. But `vhost.conf` sets `ErrorDocument 404` with no equivalent for 500, so users got Apache's stock "Internal Server Error" page: no explanation, no branding, no indication whether it was worth retrying or whether anyone knew. This didn't cause the outage and — on the evidence — didn't measurably delay the report either: the first user-visible 500 was at 13:44:38 and it was reported at ~13:47, a three-minute turnaround. The 6h29m duration is almost entirely the 07:46 → 13:44 window in which nobody hit a field-rendering page at all, which is a monitoring-coverage problem, not a comprehension one.

What the bare page cost was the *quality* of the report rather than its speed. A stock "Internal Server Error" gives the reporter nothing to relay beyond "it's broken" — no indication of which subsystem, whether it's their session, or whether it's worth retrying. That matters for triage even when, as here, the user reports it promptly anyway. (Framing corrected per lucos-ux, who pointed out the original wording claimed a delay the timeline doesn't support.)

---

## What Was Tried That Didn't Work

Nothing was tried that failed — the container log named the failing function and file on the first read, and the image comparison confirmed it immediately.

Two things are worth recording as *deliberately not* done:

- **No container restart.** Restarting would have achieved nothing: the broken code is the image, and a restart would have re-launched the same one. The reflex "restart first, diagnose second" is wrong when the failure arrived with a deploy.
- **No Dependabot `ignore` rule as the stopgap.** lucas42/lucos_media_import uses that approach for python pre-releases, but the June mail incident deliberately dropped its equivalent (lucas42/lucos_mail#62) in favour of a verified CI guard, on the grounds that a failing visible check is better signal than a PR that silently never opens. The same reasoning was applied here.

  Worth recording that this reasoning survived a check rather than being assumed. lucos-system-administrator verified the media_import ignore rule against reality: it was added 2026-06-11, its Dockerfile floats on `python:3.14`, Docker Hub has published `3.15.0b3` / `3.15.0b4` / `3.15-rc` since 2026-07-16, and no python Dependabot PR has been opened on that repo in the ~40 days since on a daily schedule. So the ignore syntax **does** work, contrary to the "may fail silently" hedge in its own code comment — an ignore rule is a legitimate mechanism, not a broken one.

  The argument against it is subtler, and is the one that generalises: **a working ignore rule is invisible in effect.** Nothing fails, nothing logs, and nothing would announce it if a registry's pre-release tag format shifted and silently stopped matching the wildcard. We only know it held because someone went and checked Docker Hub by hand. A failing CI check announces breakage; a silently-correct ignore announces nothing either way.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| **Restore:** revert base image to `php:8.5.8-apache-trixie`, plus a CI guard rejecting alpha/beta/rc base image tags | lucas42/lucos_media_metadata_manager#386 | Done — merged 14:13 UTC |
| **Build-time fix:** run CI tests against the built production image, **and** add a test that renders a view — both halves, since parity without coverage would not have caught this | lucas42/lucos_media_metadata_manager#387 | Ready — High, owner lucos-developer |
| Give `/_info` a way to notice its own render path is broken — recommendation revised (see below) to *render a field view with a fixed input and assert the expected markup*, rather than asserting an allowlist of required extensions | lucas42/lucos_media_metadata_manager#388 | **Blocked** on lucas42/lucos#273 — Medium, owner lucos-developer |
| Add `ErrorDocument 500` so failures render a human-readable page instead of Apache's default, with no dependency on the app's own bootstrap/auth/API code | lucas42/lucos_media_metadata_manager#389 | Ready — Medium, owner lucos-ux |
| **Estate-wide:** decide the convention for guarding against base-image bumps that break at runtime — third production break of this class, four different repo-local defences, no policy | lucas42/lucos#273 | Awaiting Decision — High, owner lucas42 |
| **Estate-wide `/_info` spec:** a service whose `/_info` checks only describe its *dependencies* cannot report its own failure. Owned by lucos-architect, deliberately **not** filed separately — it is the load-bearing half of the answer to lucas42/lucos#273, and splitting it risks the CI-guard half shipping alone | tracked within lucas42/lucos#273 | Awaiting the lucas42/lucos#273 decision |

**Why lucas42/lucos_media_metadata_manager#388's recommendation changed.** The report originally recommended asserting that required PHP extensions are loaded. lucos-architect argued that an extension allowlist is tuned precisely to the incident that just happened and rots: the next time code calls into an extension nobody thought to list, the check passes and the page 500s again. A check that renders a field view and asserts its markup needs no list, and fails on a missing extension, a broken include, a syntax error or a half-deployed image alike — strictly more coverage for comparable effort. That argument is correct and the recommendation is revised accordingly.

It also changes what kind of decision this is. Asserting loaded extensions sits comfortably inside the existing `/_info` contract; asserting *rendered output* extends that contract into content correctness, which is an estate-wide spec question rather than one repo's implementation detail. That is why the ticket is Blocked on lucas42/lucos#273 rather than Ready — shipping the narrower check now would likely be replaced within weeks. If the decision goes the other way, lucas42/lucos_media_metadata_manager#388 returns to Ready with the original extension-assertion approach unchanged.

The expensive option — an authenticated synthetic prober fetching real pages — remains out of scope on cost grounds under either outcome.

---

## Lessons

- **A healthy container is not a working service.** Both the Docker healthcheck and monitoring were satisfied by an endpoint that shares no code with the thing that was broken. When the only check on a service probes its *dependencies*, the service cannot report its own failure.
- **"It builds" is a weak gate for a base-image change.** All three base-image incidents recorded here built cleanly. The distinguishing question is whether anything in CI *exercises* the image, and the answer is per-repo and accidental.
- **A guard must exercise the thing it's guarding, not merely start it.** Booting the artefact and asking `/_info` whether it's well proves the artefact can answer questions. Both obvious remedies here — a CI `/_info` smoke test, and tests-in-the-shipped-image — would have gone green through this outage.
- **The quiet failures are the expensive ones.** The June mail outage was the same class of defect and lasted 14 minutes, because it crash-looped loudly. This one lasted hours because it failed politely.
- **Prefer guards that fail loudly over guards that succeed silently.** A dependabot `ignore` and a CI assertion can both stop a bad bump, but only one of them tells you it's still working. This generalises well beyond base images.

---

## Sensitive Findings

None — and the reason is slightly stronger than "the pages are access-controlled". Verified by lucos-security against the source:

- The auth gate runs **before** the code that fataled. `src/html/albums.php` calls `require_once("../authentication.php")` and `requireScope("media-metadata:read")` at line 62-63, ahead of the `viewAlbum()` call at line 98 that triggered the fatal; tracks and collections follow the same pattern. So no unauthenticated request could reach the crashing code at all — this isn't "sensitive output that happened to be behind a login", it was unreachable pre-auth.
- `php.ini-production` is genuinely in use (`RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"` in the Dockerfile) with no `display_errors` override anywhere in the repo, so no stack trace reached any browser. The fatal text stayed in the container log.
- `vhost.conf` sets only `ErrorDocument 404`, so the 500 was Apache's stock page with no application data path into it.
