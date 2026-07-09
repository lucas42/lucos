# Audit: what users see when aithne is unavailable

| Field | Value |
|---|---|
| **Date** | 2026-07-10 |
| **Trigger** | lucas42/lucos#260, raised after the ~46-minute aithne crash-loop on 2026-06-30 (`docs/incidents/2026-06-30-aithne-kek-migration-deploy-race.md`) left an open question: the incident report covers operator recovery thoroughly, but nobody checked what a *user* saw when they tried to log in during the outage. |
| **Scope** | The login / new-token-issuance error path only — what a user sees when they try to sign in and aithne cannot respond. Does not cover resilience for already-authenticated sessions in general (tracked separately in lucas42/lucos#255); JWKS serve-stale behaviour is discussed below only where it materially changes what a user sees. |
| **Method** | For every service found to call `AITHNE_ORIGIN` directly, read the code path that handles a failed aithne call, then — for every service except `lucos_worlds` — actually ran it locally with `AITHNE_ORIGIN`/`AITHNE_JWKS_URL` pointed at a connection-refused address (and, in several cases, a stub returning HTTP 500) and observed the real HTTP response a browser would receive. `lucos_worlds` (third-party BookStack + OIDC plugin) was assessed by reading the exact pinned vendor image's source rather than a full live OIDC round-trip. |

## Services covered

Found via a code search across the org for `AITHNE_ORIGIN`, covering the four named as a minimum in lucas42/lucos#260 (`lucos_creds`, `lucos_notes`, `lucos_media_seinn`, `lucos_loganne`) plus every other consumer found: `lucos_contacts`, `lucos_eolas`, `lucos_backups`, `lucos_media_metadata_manager`, `lucos_photos`, `lucos_arachne` (the `explore` sub-app), and `lucos_worlds`. Eleven services in total.

## Summary finding: one shared pattern across (almost) every consumer

Every one of the ten in-house consumer services was, independently, engineered to fail *safely* when a JWKS/token-verification call to aithne fails — none of them leak a raw exception, a framework debug page, or a stack trace to the browser. That part of the estate's engineering is solid, and several teams clearly put real thought into it (see the JWKS serve-stale rollout below).

But every one of them handles a **confirmed connectivity failure** the same way: it's silently collapsed into the generic "not authenticated" branch, and the user is issued a redirect toward `{AITHNE_ORIGIN}/auth/login?...` — i.e. straight back to aithne itself. When aithne is actually unreachable (as it was for ~46 minutes on 2026-06-30), that redirect target fails to load too, and the user's browser shows its own default "can't reach this page." lucas42/lucos#260 explicitly names this pattern as one of the failure modes to look for ("browser-default error page"), and it's what almost every service in the estate does today, whether or not the service itself crashes.

The key distinction driving each finding below: a request with **no session cookie at all** genuinely can't tell whether aithne is up before redirecting there — that's normal IdP-redirect behaviour and isn't treated as a defect here. A request **with a cookie that fails verification because the JWKS fetch itself fails on a connection error** is different: the service has first-hand, in-request knowledge that aithne is unreachable (not just "this token is invalid") and discards that signal before choosing how to respond. That's the gap every "NEEDS-FIX" finding below is about.

A second, narrower finding affects four services specifically: a bug in the shared `isJWKSInfraError` predicate (misclassifying real connection errors) that likely also undermines the JWKS serve-stale safety net built to survive brief outages.

## Per-service findings

| Service | Verdict | Live-tested? | Follow-up filed |
|---|---|---|---|
| `lucos_creds` | Needs fix (2 issues combined) | Yes | lucas42/lucos_creds#449 |
| `lucos_notes` | Needs fix (2 issues combined) | Yes | lucas42/lucos_notes#459 |
| `lucos_media_seinn` | Needs fix (2 issues combined) | Yes | lucas42/lucos_media_seinn#553 |
| `lucos_loganne` | Needs fix (2 issues combined) | Yes | lucas42/lucos_loganne#565 |
| `lucos_contacts` | Needs fix | Yes | lucas42/lucos_contacts#772 |
| `lucos_eolas` | Needs fix | Yes | lucas42/lucos_eolas#333 |
| `lucos_backups` | Needs fix (partial mitigation for warm sessions) | Yes | lucas42/lucos_backups#369 |
| `lucos_media_metadata_manager` | Needs fix (partial mitigation for warm sessions) | Yes | lucas42/lucos_media_metadata_manager#367 |
| `lucos_photos` | Needs fix (partial mitigation for warm sessions) | Yes | lucas42/lucos_photos#468 |
| `lucos_arachne` (`explore`) | Needs fix (partial mitigation for warm sessions) | Yes | lucas42/lucos_arachne#728 |
| `lucos_worlds` (BookStack + OIDC) | Needs fix — different shape (raw text in an otherwise well-behaved toast) | No — code-read of the pinned vendor image | lucas42/lucos_worlds#51 |

### `lucos_creds`, `lucos_notes`, `lucos_media_seinn`, `lucos_loganne` — the four JS consumers with `isJWKSInfraError`

These four picked up an identical copy of a `isJWKSInfraError(error)` predicate from the same estate-wide JWKS-serve-stale rollout (lucas42/lucos_aithne#241, landed via lucas42/lucos_creds#447, lucas42/lucos_notes#455, lucas42/lucos_media_seinn#552, lucas42/lucos_loganne#564). The predicate checks `error.code === 'ECONNREFUSED'` etc. directly — but `jose`'s `createRemoteJWKSet` uses Node's native `fetch`, which wraps connection failures as `TypeError('fetch failed', { cause: <original error with .code> })`. The real code lands on `error.cause.code`, not `error.code`. Confirmed live in all four: a genuine connection-refused logs as an ordinary `JWT verification failed: fetch failed`, never the intended infra-error path — which is also the gate for the serve-stale fallback, so that safety net plausibly doesn't engage for the exact failure mode it exists to survive.

All four also exhibit the shared redirect-to-dead-aithne pattern described above. `lucos_notes` already has a local error-page template (`page.ejs`, used for its 403 case) that's a ready-made target for the fix; the others would need to add one.

Filed as one combined issue per repo (fix classification + add local interstitial), since the two problems are closely linked — a correct classification is a prerequisite for reliably triggering the local-page fix.

### `lucos_contacts`, `lucos_eolas` — shared Django `lucosauth` lineage

Both catch the JWKS connection failure internally (logged as "failing closed") but silently treat the request as an ordinary anonymous visitor, then redirect. No Django debug page, no exception — but the specific "aithne is unreachable" signal never reaches the user. Near-identical code between the two apps, but each has its own copy of `lucosauth` and needs its own PR.

### `lucos_backups`, `lucos_media_metadata_manager`, `lucos_photos`, `lucos_arachne` (`explore`) — independent JWKS-caching implementations, same cold-start gap

Each of these four built its own last-known-good/resilient JWKS caching layer (`_LKGJWKSClient` in Python, a `$cachedRawJwks` field in PHP, `_ResilientJWKSClient` in Python, `_createLKGJWKSet` in JS) — genuinely good engineering that likely shields an already-active session from a brief outage, provided its cache is still warm. None of them render a local message for the cold-start case: a new session, or any session whose cache window has lapsed during a longer outage (the real incident ran ~46 minutes — longer than every observed cache TTL). That case still hits the same blind redirect-to-dead-aithne pattern.

### `lucos_worlds` — different shape: a raw exception string in an otherwise well-designed toast

BookStack's OIDC plugin already does the right structural thing on a discovery failure — it catches the exception and shows a branded, HTML-escaped toast on the login page rather than crashing. But the toast's *content* is the raw exception message verbatim (e.g. "OIDC Discovery Error: cURL error 7: Failed to connect to aithne.l42.eu port 443: Connection refused"), which fails the "plain English" and "explicit retry guidance" bars just as much as a broken page would. lucos already has a working patch mechanism for this vendor code (ADR-0002), so the fix is a small, well-scoped patch rather than new infrastructure.

## What "acceptable" would look like

Per lucas42/lucos#260's own framing, and applied consistently across every finding above: when a service can tell, in-request, that the failure is aithne being unreachable — not merely "this token doesn't verify" — it should render its own plain-English page rather than redirect toward a server it just confirmed is down. Suggested copy, consistent with `~/.claude/agent-memory/lucos-ux/copy_error_retry_guidance.md`:

> Sign-in is temporarily unavailable. Please try again in a few minutes.

Prose in `<p>`, any technical detail (if shown at all, e.g. for operators) in a separate `<pre>`.

## What's *not* being asked for

None of the eleven follow-up issues ask a service to add a proactive aithne health-check before every unauthenticated redirect — that would add latency to the normal case for no real benefit, since a never-logged-in user redirected toward a down aithne is looking at aithne's own failure surface, not the consumer's. The fix in every case is scoped narrowly to the moment the service *already* knows, from its own failed network call, that aithne is unreachable.

## A note on how to route the follow-ups

All eleven issues linked above are filed and on the board for triage, each scoped to a single repo since the fix (however similar in shape) has to land as separate code in separate languages/frameworks. Given how consistently the same pattern showed up, it may be worth team-lead/architect considering whether to dispatch these as a coordinated batch rather than as eleven independent, unrelated-looking tickets — that's a coordination call outside this audit's scope, so it's flagged here rather than decided.

Every issue is tagged as mixed backend (error classification/control flow) + frontend (copy/template) work, with a suggested routing to `lucos-developer` with a UX consult on copy, rather than `lucos-ux` alone — consistent with how this repo's persona conventions split mixed-scope work.
