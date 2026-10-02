# Incident: lucos_worlds page views 500 after BookStack 26.09 bump

| Field | Value |
|---|---|
| **Date** | 2026-10-01 |
| **Duration** | Broken from 2026-09-29 08:40 UTC (deploy). First failed page view 2026-10-01 20:57 UTC. Fixed 2026-10-02 00:43 UTC: 3h46m of user-visible outage |
| **Severity** | Complete outage of page viewing (the service's core read path). Every page returned BookStack's generic "An Error Occurred" page with a return-home link and no explanation. Login, book and chapter listings, and `/_info` still worked (all 200 in the access log during the outage) |
| **Services affected** | lucos_worlds |
| **Detected by** | User report (lucas42). Monitoring stayed green throughout |

---

## Summary

A Dependabot bump of the BookStack base image (26.05.5 → 26.09) was auto-merged and deployed on 2026-09-29. BookStack 26.09 refactored the page view's sidebars, and lucos_worlds overrides that view with a whole-file patched copy written against the older version. From that deploy on, every page view returned HTTP 500 (`Undefined variable $pageNav`). Nobody opened a page for about 2.5 days, so the outage was first hit, and reported, on the evening of 2026-10-01. The fix (lucas42/lucos_worlds#97) re-bases the patched view on upstream 26.09. The fix was deployed at 00:43 UTC on 2026-10-02 and confirmed by production page views returning 200 at 00:54, and by lucas42. The 26.09 sidebar sections (page nav, book tree, page actions) were checked present in the local render test. On production they were confirmed only by lucas42's report that pages load; keyboard reachability of the new view-block sidebars was not checked.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-07-12 | `patches/views/pages/show.blade.php` added (lucas42/lucos_worlds#52): a whole-file copy of upstream's view with one changed line (og:description excerpt) |
| 2026-09-28 21:xx | Last successful page view in the nginx access log |
| 2026-09-29 08:34 | Dependabot opens lucas42/lucos_worlds#95, bumping `linuxserver/bookstack` 26.05.5 → 26.09.20260929 |
| 2026-09-29 08:36 | lucas42/lucos_worlds#95 auto-merged |
| 2026-09-29 08:40 | New `lucos_worlds_web` container (v1.2.30) starts. Every page view is now broken, but there is no traffic to show it |
| 2026-10-01 20:57 | First page view since the deploy returns 500. All 15 page GETs between here and 00:32 on 2026-10-02 return 500 |
| 2026-10-02 ~00:30 | lucas42 reports 500s on page view. SRE investigation begins |
| 2026-10-02 ~00:35 | Root cause identified from laravel.log, the running container's source, and upstream's diff |
| 2026-10-02 00:38 | Controlled local reproduction: `origin/main` image → 500 with the identical exception; fixed image → 200 |
| 2026-10-02 00:39 | lucas42/lucos_worlds#97 opened |
| 2026-10-02 00:40 | lucos-code-reviewer approves (00:40:46). On this unsupervised repo that is the merge trigger, and lucas42/lucos_worlds#97 merges at 00:40:59 |
| 2026-10-02 00:43 | `lucos_worlds_web` 1.2.31 starts. Deployed `show.blade.php` confirmed byte-identical (sha256) to the tested file; no laravel errors after 00:32:38 |
| 2026-10-02 00:54 | First production page views after the fix (`/books/kaidoho/page/monster-template`, `/books/kaidoho/page/session-0`) return 200, with no new laravel errors. lucas42 confirms pages load again. Incident resolved |

---

## Analysis

### Root cause: whole-file patch of a view upstream refactored

lucos_worlds customises BookStack by `COPY`ing whole-file replacements over upstream source in the Dockerfile. There are five of them. `patches/views/pages/show.blade.php` changes a single line (`og:description` uses `Page::getExcerpt()`), but carries the rest of upstream's 2026-07 view with it.

BookStack v26.09 moved the page-view sidebars into "view blocks":
- `PageController::show()` no longer passes `$pageNav`, `$sidebarTree` or `$watchOptions`. A new `PagesShowPageNav` view block computes the page nav itself.
- Upstream's view swapped six sidebar `@include`s for two `@include('common.view-blocks', …)` calls.

Our stale copy still referenced the removed variables, so rendering failed on the first one:

```
ViewException: Undefined variable $pageNav (View: /app/www/resources/views/pages/show.blade.php)
```

Confirmed by: the exception logged for each production 500 (matched one-for-one against the nginx access log), and a controlled reproduction with the `origin/main` image against a fresh database, which gave the same exception. The fixed image rendered the same page with 200.

### Contributing factor: the version number hid a feature release

The base-image bump went through Dependabot auto-merge without a human looking at it, which is the intended design for minor/patch bumps. However, BookStack versions are calendar-based: 26.05.5 → 26.09 is a new feature release, not a patch, even though Dependabot's minor-and-patch grouping treated it as one. That release is the one that refactored the page view.

### Contributing factor: CI was looking, but at the wrong surface

CI did run against the real patched image on lucas42/lucos_worlds#95. All four test jobs (`test-info-endpoint`, `test-oidc-alg-binding`, `test-oidc-es256`, `test-page-excerpt`) passed before the auto-merge, so the failure was coverage, not absence.

A whole-file patch can break in three places:

1. **Our hunk.** The existing tests cover this: `test-page-excerpt` unit-tests `Page::getExcerpt()`.
2. **The rest of upstream's file**, frozen when we copied it. Only an upstream-hash guard (lucas42/lucos_worlds#98) covers this.
3. **Unpatched upstream code our frozen copy depends on**, here the controller that stopped passing `$pageNav`. Only a render test (lucas42/lucos_worlds#99) catches this.

This incident hit both 2 and 3. The Dockerfile comment saying `test-page-excerpt` stops a Dependabot bump from "silently regress[ing]" this was true only of 1. No job requested a page.

So lucas42/lucos_worlds#98 and lucas42/lucos_worlds#99 are complementary, not alternatives. The guard covers the remainder of *every* patched file, whatever any one test happens to exercise. lucas42/lucos_worlds ADR-0002 currently names integration tests as "the sole defence (lucas42's mandate)" against upgrade breakage, so the guard adds a second defence alongside his decision and needs his agreement.

### Contributing factor: other patches drifted too

In the same bump, two other patched files also changed upstream, and our copies silently revert those changes:
- `Page.php`: the drift is docblock-only.
- `OidcJwtWithClaims.php`: upstream removed one line, `array_filter($parsedKeys)`. lucos-security reviewed it and found the retained line is a no-op, since every entry is a key object, and the `alg()` check still runs. `test-oidc-es256` also passed on lucas42/lucos_worlds#95.

The other two OIDC patch targets (`OidcProviderSettings.php`, `OidcJwtSigningKey.php`) are byte-identical upstream between v26.05.5 and v26.09, so they did not drift in this bump. Three of the five patches sit in the OIDC login/token-verification path, so a future upstream verification fix there would be silently reverted too. The guard therefore has security value as well as reliability value.

### Contributing factor: detection depended on a human opening a page

`/_info` checks BookStack's dependencies (database, cache, session), not rendering, so it stayed green. That is consistent with `/_info`'s availability-not-correctness boundary, and build-time checks (lucas42/lucos_worlds#98, lucas42/lucos_worlds#99) catch this earlier and more cheaply, so no runtime render check is proposed for this repo. The estate-level question of runtime breakage that CI and `/_info` both miss is open in lucas42/lucos#273, and this incident is a fifth base-image data point for it. The low traffic turned a deploy-time break into a 2.5-day latent one. That cost nothing here, because nobody needed a page in that window, but it means the "breaking change" and the "outage" were 60 hours apart. Anyone working backwards from the report time would have looked at the wrong deploy.

---

## What Was Tried That Didn't Work

- A container restart was not attempted. The fault is in the image, so a restart would just serve the same broken view again.
- The first local reproduction run hit an unrelated 500 (`Directory …/cache/purifier/URI not writable`). That was a test-harness artefact: running `artisan tinker` as root created BookStack's purifier cache directory owned by root. Running tinker as the app user (`abc`) fixed it. Noted for anyone reusing the recipe.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Re-base patched `show.blade.php` on BookStack v26.09 | lucas42/lucos_worlds#97 | Done |
| Build-time upstream-hash guard on all whole-file patches, plus re-basing the drifted patches (OIDC first). Needs lucas42's agreement, since it adds a second defence alongside ADR-0002's tests-only mandate. lucos-architect will then write a lucos_worlds ADR for the patch-carrying policy | lucas42/lucos_worlds#98 | Awaiting Decision (lucas42) |
| Estate convention for runtime breakage that CI and `/_info` both miss (this incident is a fifth data point) | lucas42/lucos#273 | Open |
| CI render smoke test: build the image, create a page, GET it, assert 200. A per-repo instance of what lucas42/lucos#273 is deciding estate-wide | lucas42/lucos_worlds#99 | Ready (lucos-developer) |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
[ ] Yes — see note below.
