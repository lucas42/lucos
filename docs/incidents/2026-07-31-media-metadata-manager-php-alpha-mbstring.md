# Incident: media-metadata.l42.eu 500s — PHP alpha base image dropped mbstring

| Field | Value |
|---|---|
| **Date** | 2026-07-31 |
| **Duration** | Broken from 07:46 UTC, restored 14:15 UTC (~6h29m). User-visible impact from the first 500 at 13:44 UTC (~31 minutes) |
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
| 14:04 | Hotfix PR lucas42/lucos_media_metadata_manager#386 opened (revert base image + CI guard against pre-release tags) |
| 14:13:14 | PR approved by lucos-code-reviewer and lucas42, auto-merged; deploy pipeline 985 starts |
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

This is `/_info` behaving as specified — availability and dependency config, not content-rendering correctness — but the practical consequence is that a total user-facing outage of this service was invisible. The six-hour delay was luck, not detection: the breakage started at 07:46 and was only noticed when someone happened to browse at 13:44.

### Aggravating factor: the 500 page said nothing

PHP runs with `php.ini-production`, so `display_errors` is off — correct for production. But `vhost.conf` sets `ErrorDocument 404` with no equivalent for 500, so users got Apache's stock "Internal Server Error" page: no explanation, no branding, no indication whether it was worth retrying or whether anyone knew. This didn't cause the outage, but it made it harder for a user to recognise as reportable, and plausibly contributed to the delay in it being raised.

---

## What Was Tried That Didn't Work

Nothing was tried that failed — the container log named the failing function and file on the first read, and the image comparison confirmed it immediately.

Two things are worth recording as *deliberately not* done:

- **No container restart.** Restarting would have achieved nothing: the broken code is the image, and a restart would have re-launched the same one. The reflex "restart first, diagnose second" is wrong when the failure arrived with a deploy.
- **No Dependabot `ignore` rule as the stopgap.** lucas42/lucos_media_import uses that approach for python pre-releases, but the June mail incident deliberately dropped its equivalent (lucas42/lucos_mail#62) in favour of a verified CI guard, on the grounds that a failing visible check is better signal than a PR that silently never opens. The same reasoning was applied here.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| **Restore:** revert base image to `php:8.5.8-apache-trixie`, plus a CI guard rejecting alpha/beta/rc base image tags | lucas42/lucos_media_metadata_manager#386 | Done — merged 14:13 UTC |
| **Durable fix:** run CI tests against the built production image, and add a test that actually renders a view | lucas42/lucos_media_metadata_manager#387 | Open |
| Give `/_info` a way to notice its own render path is broken (recommended: assert required PHP extensions are loaded) | lucas42/lucos_media_metadata_manager#388 | Open |
| Add `ErrorDocument 500` so failures render a human-readable page instead of Apache's default | lucas42/lucos_media_metadata_manager#389 | Open |
| **Estate-wide:** decide on a convention for guarding against auto-merged base-image bumps — third production break of this class, four different repo-local defences, no policy | lucas42/lucos#273 | Open — needs a decision from lucas42 / the architect |

---

## Lessons

- **A healthy container is not a working service.** Both the Docker healthcheck and monitoring were satisfied by an endpoint that shares no code with the thing that was broken. When the only check on a service probes its *dependencies*, the service cannot report its own failure.
- **"It builds" is a weak gate for a base-image change.** Every base-image incident in this estate so far has built cleanly. The distinguishing question is whether anything in CI runs the image, and the answer is per-repo and accidental.
- **The quiet failures are the expensive ones.** The June mail outage was the same class of defect and lasted 14 minutes, because it crash-looped loudly. This one lasted hours because it failed politely.

---

## Sensitive Findings

None. No credentials, personal data, or security-relevant material were involved — the failing pages are behind aithne authentication and the fatal error text was confined to the container log.
