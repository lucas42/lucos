# Incident: lucos_locations map UI down — oauth2-proxy crash-loop on missing prod credentials

| Field | Value |
|---|---|
| **Date** | 2026-07-09 |
| **Duration** | ~51 minutes (23:42 UTC to 00:33 UTC) |
| **Severity** | Partial degradation (human map UI only; device ingestion unaffected) |
| **Services affected** | lucos_locations (human map UI: `/map`, `/owntracks/api\|ws\|view\|static\|utils`) |
| **Detected by** | User report (lucas42) |

Source incident: reported via team-lead; go-live tracked on lucas42/lucos_locations#96, feature PR lucas42/lucos_locations#97.

---

## Summary

lucos_locations PR lucas42/lucos_locations#97 fronted the human map-UI paths with an oauth2-proxy sidecar (nginx `auth_request`), gated on the aithne `locations:read` scope. It merged and auto-deployed to production at 23:42 UTC **before** the prod credentials the sidecar needs had been set. This was a *known, foreseen* gap rather than a surprise: those creds are lucas42-only and were an explicitly-tracked outstanding dependency (lucas42/lucos_locations#96), and team-lead flagged at merge time that auto-deploying ahead of them would bring the sidecar up misconfigured and break the human map UI. The oauth2-proxy container therefore crash-looped on startup (`missing setting: cookie-secret / client-secret`), and every request to a gated path returned **500** because nginx was `auth_request`-ing a dead sidecar. The underlying Go app was healthy throughout (`/_info` → 200), and the device publish path + MQTT — which use separate credential auth — were unaffected, so no location data was lost. Resolved by setting the prod creds (lucas42) and redeploying both aithne (to register the new OIDC client at startup) and lucos_locations (to inject the sidecar's creds).

---

## Timeline

| Time (UTC) | Event |
|---|---|
| (pre-merge) | The lucas42-only prod linked credential was a known outstanding dependency, tracked on lucas42/lucos_locations#96 since 2026-07-07 |
| ~23:42 (at merge) | team-lead flags that the prod creds are not yet set and an auto-deploy will bring the sidecar up misconfigured and break the human map UI |
| 23:42:21 | PR lucas42/lucos_locations#97 merged to main |
| 23:42:23 | Auto-deploy (CircleCI pipeline 193) starts; oauth2-proxy sidecar comes up without its prod creds |
| ~23:43–23:56 | oauth2-proxy begins crash-looping: `invalid configuration: missing setting: cookie-secret / client-secret`; `/map/` returns 500 |
| ~00:0X | lucas42 reports 500 on https://locations.l42.eu/map/ |
| ~00:08–00:26 | SRE investigation: root cause confirmed by direct container observation (oauth2-proxy `Restarting`, exit=1, restart count 31); device split verified intact (`/owntracks/pub` → 401, MQTT :8883 open) |
| ~00:2X | lucas42 sets prod creds (linked cred lucos_locations/production ⇒ lucos_aithne/production; AITHNE_ORIGIN/JWKS_URL/TOKEN_URL; OAUTH2_PROXY_COOKIE_SECRET) |
| 00:29:51 | aithne redeployed (pipeline 542); startup reconcile logs `upserted client "lucos_locations"` — OIDC client now registered |
| 00:30:23 | lucos_locations redeployed (pipeline 195) to inject sidecar creds |
| 00:33:05 | oauth2-proxy comes up healthy (`restarts=0`, no more crash-loop); `/map/` now serves the oauth2-proxy sign-in page (see verification note) instead of 500; incident resolved |
| ~00:35 | Post-change verification: monitoring 54/54 (matches baseline), device path 401 + MQTT open, aithne `/_info` 200 |

---

## Analysis

### Root cause: gated deploy shipped ahead of its prod credentials

PR lucas42/lucos_locations#97 introduced an oauth2-proxy sidecar in front of the human map-UI paths. The sidecar requires prod credentials to start: an OIDC `client-secret` (`KEY_LUCOS_AITHNE`, delivered by the prod linked credential lucos_locations/production ⇒ lucos_aithne/production), a `cookie-secret` (`OAUTH2_PROXY_COOKIE_SECRET`), and the aithne endpoint URLs. These are production credentials, which only lucas42 can write, and they were still outstanding (tracked on lucas42/lucos_locations#96) when the PR merged and auto-deployed.

Critically, this dependency was **known and the failure was foreseen** — not an unnoticed gap. The prod linked credential had been an explicitly-tracked outstanding item on lucas42/lucos_locations#96 since 2026-07-07, and team-lead flagged at merge time that an auto-deploy ahead of the creds would bring the sidecar up misconfigured and break the human map UI. The incident occurred because the merge/deploy of a credential-gated change was not sequenced to *follow* the setting of its lucas42-only prod creds. That sequencing — not a missing detection mechanism alone — is the primary prevention lever (see Follow-up Actions).

With no `cookie-secret` and no `client-secret`, oauth2-proxy exits at startup with `invalid configuration` and crash-loops. nginx (`lucos_locations_otfrontend`, healthy) is configured to `auth_request` the sidecar for every human path; a dead sidecar means the subrequest fails and nginx returns **500** to the client. This was confirmed by direct observation, not log-reading alone: the container was in `Restarting` state with `exit=1` and a climbing restart count (31 at time of diagnosis).

### Why the blast radius was contained

The design deliberately split human and device auth. The device publish path (`/owntracks/pub`) and MQTT (`:8883`) authenticate with a separate basic-auth / mosquitto credential, entirely independent of oauth2-proxy. Throughout the incident `/owntracks/pub` returned **401** (not 500) and MQTT `:8883` stayed open, so devices continued recording location fixes — the `location-freshness` check stayed green and no data was lost. Only the interactive map UI was affected.

### Why `/_info` did not catch it

`/_info` is served by the Go app (otrecorder), which was healthy the entire time — so monitoring stayed green (54/54) while the map UI was down. The outage lived entirely in the auth sidecar sitting *in front of* the app. This is a general property of sidecar-fronted services: an app-level health endpoint cannot see a failure in a proxy layer ahead of it.

### Why two redeploys were needed to resolve

Setting the creds in lucos_creds is not sufficient on its own — running containers hold the environment from their last `up`, and a bare `docker restart` does not re-fetch creds. Two fresh deploys were required:

1. **aithne** — per ADR-0004, aithne reconciles its `oidc_clients` store from `CLIENT_KEYS` **only at startup** (`reconcileOIDCClients`, called once from `main()`). Before the linked cred existed, aithne's reconcile log-and-skipped the client (`no CLIENT_KEYS secret for declared client "lucos_locations" — skipping`). A redeploy was needed for aithne to register the owntracks OIDC client; without it the token exchange would fail even with the sidecar up.
2. **lucos_locations** — a fresh deploy to inject the sidecar's now-present creds into the container environment.

---

## Resolution & Verification

After both redeploys, verified end-to-end (headless — a real logged-in 200 is a browser check for lucas42):

- **oauth2-proxy healthy** — `Up`, `restarts=0`, log shows `OAuthProxy configured for OpenID Connect Client ID: lucos_locations`. Crash-loop gone; no more 500.
- **OIDC wiring correct** — `/oauth2/start` → **302** to `https://aithne.l42.eu/oauth2/authorize?client_id=lucos_locations&scope=openid+email+profile+locations:read&redirect_uri=https://locations.l42.eu/oauth2/callback&…`. Correct client, scope, and callback.
- **Device split intact** — `/owntracks/pub` → 401, MQTT `:8883` open, `/_info` → 200, `location-freshness` green.
- **Monitoring** — 54/54 healthy, matching the pre-change baseline; the aithne redeploy caused no estate auth regression.

**Verification note — `/map/` returns 403, not a 302 redirect.** An unauthenticated `/map/` serves the **oauth2-proxy sign-in page with HTTP status 403**, not a redirect to aithne. This is the *configured* behaviour of PR lucas42/lucos_locations#97's nginx, which uses `error_page 401 =403 /oauth2/sign_in` on the gated locations — a 401 from the `auth_request` is rewritten to 403 and served the sign-in-button page (which links to `/oauth2/start` → aithne). A human therefore gets a working "Sign In" prompt, not a dead error — the go-live is functional. Whether the intended UX is this intermediate button page or an automatic bounce to aithne (`skip-provider-button` + `error_page 401 = /oauth2/start`) is a UX decision for lucas42, flagged below, not an incident.

---

## What Was Tried That Didn't Work

No dead ends — root cause was confirmed on first investigation and the resolution worked as planned. One thing worth recording for future responders: a host-level `docker compose restart` / `docker restart` would **not** have fixed this, because it reuses the container's existing environment and does not re-fetch the newly-set creds from lucos_creds. A fresh CI deploy is required to inject new credentials.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Sequence lucas42-only prod creds *before* merging/deploying a credential-gated change, and/or add a guard so such a deploy can't silently ship a crash-looping sidecar | #266 | Awaiting decision (lucas42) |
| Decide intended `/map/` unauth UX: keep the 403 sign-in-button page, or auto-bounce to aithne (`skip-provider-button`) | lucas42/lucos_locations#99 | Awaiting decision (lucas42) |

**Recommendation under consideration:** the primary lever is a **process** one, because the failure was foreseen — when a credential-gated deploy is known in advance, set the lucas42-only prod creds *before* the merge/auto-deploy rather than after, so the sidecar never comes up misconfigured. A **detection** guard reinforces this as a backstop: a compose-level healthcheck/dependency that fails the deploy loudly, or a checklist gate on the go-live issue, so that if the sequencing is missed the deploy fails visibly instead of silently shipping a crash-looping sidecar. To be weighed against effort — impact here was contained + self-evident from the crash-loop, and recoverable in one redeploy once creds were set. team-lead is confirming with lucas42 whether to file this as a tracked reliability issue.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

Credentials are referenced by **name** only (`KEY_LUCOS_AITHNE`, `OAUTH2_PROXY_COOKIE_SECRET`, the linked credential pairing). No secret values appear in this report or were exposed by the incident. The oauth2-proxy failure was fail-closed (500, no access granted), not fail-open.
