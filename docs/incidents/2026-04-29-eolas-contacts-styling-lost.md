# Incident: eolas and contacts UIs unstyled for ~6.5 hours after volume cleanup exposed a build-time `collectstatic` bug

| Field | Value |
|---|---|
| **Date** | 2026-04-29 |
| **Duration** | ~6 hours 29 minutes (eolas: 16:14 UTC → 22:43 UTC; contacts: 16:14 UTC → 22:48 UTC) |
| **Severity** | Partial degradation — services functional but UIs unusable (every Django admin page rendered as unstyled HTML) |
| **Services affected** | `lucos_eolas` (eolas.l42.eu), `lucos_contacts` (contacts.l42.eu) |
| **Detected by** | User report from lucas42 |

Source issues: [`lucas42/lucos_eolas#217`](https://github.com/lucas42/lucos_eolas/issues/217), [`lucas42/lucos_contacts#671`](https://github.com/lucas42/lucos_contacts/issues/671)

---

## Summary

Earlier in the day, two PRs landed that consolidated each service's Docker build and moved `collectstatic` from container startup into the image build step (`lucas42/lucos_eolas#213`, `lucas42/lucos_contacts#669`). Both PRs introduced a new minimal `settings_collectstatic.py` that omitted `django.contrib.admin` from `INSTALLED_APPS`, so the build-time `collectstatic` silently skipped the entire Django admin static tree. The breakage was masked at first because the previously-mounted `*_staticfiles` named volumes still contained the old (correctly-collected) admin assets. Once those orphaned volumes were removed (the planned follow-up step from `lucas42/lucos_eolas#214` and `lucas42/lucos_contacts#670`), nginx began serving 404 for every admin CSS/JS/SVG path on both services. The fix on each side was a 4-line change adding `contenttypes`, `auth`, and `admin` to `INSTALLED_APPS` in `settings_collectstatic.py`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 13:30 | `lucas42/lucos_contacts#669` merges. New build-time `collectstatic` ships in `lucos_contacts:v1.0.28` — admin static files absent from the image, but masked by the `lucos_contacts_staticfiles` named volume (still mounted, still containing the old correctly-collected assets from the 2026-03-20 first-init). |
| 16:05 | `lucas42/lucos_eolas#213` merges. Same change ships in `lucos_eolas:v1.0.28`. Same masking applies via the `lucos_eolas_staticfiles` volume. |
| ~16:14 | SRE (this agent) executes the planned cleanup steps from `lucas42/lucos_contacts#670` and `lucas42/lucos_eolas#214`, removing the orphaned `lucos_contacts_staticfiles` and `lucos_eolas_staticfiles` named volumes on `avalon`. Nginx falls back to the image-baked `/usr/share/nginx/html/resources/`. **Both UIs begin returning 404 for every admin asset.** Monitoring's `fetch-info` check is `/_info`-based and does not exercise CSS, so no alert fires. |
| 22:25 | lucas42 notices that `eolas.l42.eu` has lost its styling and reports to team-lead. |
| 22:27 | SRE investigation starts. Probes confirm `/resources/lucos_navbar.js` returns 200 but `/resources/admin/css/base.css` returns 404. |
| 22:30 | Root cause identified: `settings_collectstatic.py` omits `django.contrib.admin` from `INSTALLED_APPS`. Reproduced locally — `docker build --target app` produces an image with no `admin/` subtree. |
| 22:32 | `lucas42/lucos_eolas#217` filed (P1) and `lucas42/lucos_eolas#218` opened with the four-line fix. |
| 22:33 | lucos-code-reviewer approves PR #218. |
| 22:35 | PR #218 merged via the `code-reviewer-auto-merge.yml` workflow following lucas42's approval. |
| ~22:38 | Team-lead surfaces a heads-up that `lucos_contacts` may have the same bug. |
| 22:40 | Verified `contacts.l42.eu/resources/admin/css/base.css` returns 404. Same `INSTALLED_APPS` bug present in `lucos_contacts/app/settings_collectstatic.py`. |
| 22:41 | `lucas42/lucos_contacts#671` filed (P1) and `lucas42/lucos_contacts#672` opened with the same four-line fix. |
| 22:43 | `lucos_eolas:v1.0.29` deploys to avalon. `eolas.l42.eu/resources/admin/css/base.css` returns 200. eolas styling restored. |
| 22:43 | PR #672 merged. |
| 22:48 | `lucos_contacts:v1.0.30` deploys to avalon. `contacts.l42.eu/resources/admin/css/base.css` returns 200. contacts styling restored. |

---

## Analysis

### Stage 1 — The latent build bug

Both services run the Django admin as their primary UI, so all of their styling comes from `django.contrib.admin`'s bundled static files. `lucas42/lucos_contacts#669` and `lucas42/lucos_eolas#213` moved `collectstatic` from container startup (using the full `settings.py`) into the Docker build step (using a new minimal `settings_collectstatic.py`). The intent was reasonable — running `collectstatic` at build time fails loudly if the project's static configuration is broken, rather than silently producing a half-empty volume on first boot.

The minimal settings file declared:

```python
INSTALLED_APPS = [
    'django.contrib.staticfiles',
]
```

Django's `collectstatic` only walks the static directories of apps listed in `INSTALLED_APPS`. Without `django.contrib.admin` (and its prerequisites `auth` and `contenttypes`), the entire `admin/` subtree was silently omitted from `STATIC_ROOT`. The build succeeded with no warnings; the image just had no admin CSS, JS, or SVG icons.

`lucas42/lucos_eolas#213`'s body explicitly named `lucas42/lucos_contacts#669` as its reference implementation, which is how the same omission propagated from contacts to eolas. A defect in a "reference implementation" is the worst kind: each subsequent copy is a confidence-multiplier rather than a fresh review.

### Stage 2 — The named volume that masked the bug

Both services had been carrying a `*_staticfiles` named Docker volume since the build-time-collectstatic-on-image-update bug from 2026-03-20 (`lucas42/lucos_eolas#212`, `lucas42/lucos_contacts#561`). That volume was first-init'd back when `collectstatic` ran inside the container at startup using the full `settings.py`, so it contained a complete admin asset tree. While the volume was still mounted on top of the image's static directory, the broken build was invisible — nginx happily served the (stale-but-complete) volume contents.

This is the named-volume-shadows-image-contents pattern: once Docker initialises a named volume from the image's contents on first creation, it never refreshes the volume from later images. Subsequent deploys serve the original contents indefinitely, regardless of what the new image actually contains. The bug was already shipped at 13:30 (contacts) and 16:05 (eolas), but had no observable effect until the masking volumes were removed.

### Stage 3 — Removing the volumes exposed the bug

Issues `lucas42/lucos_eolas#214` and `lucas42/lucos_contacts#670` were filed by the implementer of the consolidation work to track the post-deploy cleanup of the now-orphaned volumes. The SRE performed both removals around 16:14 UTC (the standard "remove the volume so it stops shadowing the image" follow-up from this kind of refactor). Once the volume was gone, the next request for `/resources/admin/css/base.css` fell through to the image — which didn't have it — and nginx returned 404.

Critically, every other check that might have caught this passed:

- The `web` container kept passing its healthcheck (the image's nginx still starts).
- Monitoring's `fetch-info` check polls `/_info`, which is a JSON endpoint served by the `app` container — it doesn't request any static asset.
- The CI build succeeded and CI's smoke tests don't load the rendered admin UI.

So the failure was effectively undetectable to anything except a human looking at the rendered page in a browser. The only thing that did detect it was lucas42's eyeballs, ~6.5 hours later.

### Stage 4 — Resolution

The fix on each side was four lines added to `settings_collectstatic.py`:

```python
INSTALLED_APPS = [
    'django.contrib.contenttypes',
    'django.contrib.auth',
    'django.contrib.admin',
    'django.contrib.staticfiles',
]
```

Verified locally on both services that `docker build --target web` now produces an image where `/usr/share/nginx/html/resources/admin/css/base.css` exists. Once the deploys landed, production probing confirmed 200 OK on the admin asset path on both hosts.

---

## What Was Tried That Didn't Work

Nothing — once the diagnostic probe of `/resources/admin/css/base.css` returned 404 while `/resources/lucos_navbar.js` returned 200, the failure mode was obvious: project-owned static files were being collected and admin-app-owned static files were not. From there the path to `INSTALLED_APPS` was direct.

Worth noting one mis-step in coordination: the SRE initially told team-lead that PR #218 would need lucas42 to merge manually because the PR had `auto_merge: null` and the visible `reusable/auto-merge` check was `skipped`. That advice was wrong — those signals are about the Dependabot auto-merge path, not the `code-reviewer-auto-merge.yml` workflow which is the relevant one for code-reviewer approvals on supervised repos. The PR auto-merged on approval. This is documented as a follow-up persona update.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix admin static asset omission on lucos_eolas | [`lucas42/lucos_eolas#218`](https://github.com/lucas42/lucos_eolas/pull/218) | Done |
| Fix admin static asset omission on lucos_contacts | [`lucas42/lucos_contacts#672`](https://github.com/lucas42/lucos_contacts/pull/672) | Done |
| Add CI assertion that admin static files exist in the `lucos_eolas` web image | [`lucas42/lucos_eolas#219`](https://github.com/lucas42/lucos_eolas/issues/219) | Open |
| Add CI assertion that admin static files exist in the `lucos_contacts` web image | [`lucas42/lucos_contacts#673`](https://github.com/lucas42/lucos_contacts/issues/673) | Open |
| Update `lucos-site-reliability` persona instructions: do not infer "needs manual merge" from `auto_merge: null` or a skipped `reusable/auto-merge` check; verify by checking for `.github/workflows/code-reviewer-auto-merge.yml` instead | `lucas42/lucos_claude_config@aab1bec` | Done |
| Update `lucos-code-reviewer` persona instructions: same auto-merge misconception, applied independently | `lucas42/lucos_claude_config@f56afde` | Done |
| Add a runtime UI-integrity check to `lucos_monitoring` for services whose primary interface is a rendered HTML UI (e.g. eolas, contacts) — detects 404s on canonical static assets and missing `<link rel="stylesheet">` references in the landing page. Came out of post-incident discussion with `lucos-ux`. | [`lucas42/lucos_monitoring#207`](https://github.com/lucas42/lucos_monitoring/issues/207) | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

- [x] No — nothing in this report has been redacted.
- [ ] Yes — see note below.
