# Incident: aithne scratch image had no CA bundle — outbound HTTPS to contacts failed

| Field | Value |
|---|---|
| **Date** | 2026-06-12 |
| **Duration** | ~1h55m (13:11 UTC broken path deployed → 15:06 UTC fix deployed; verified 15:09 UTC) |
| **Severity** | Partial degradation (admin-only) |
| **Services affected** | lucos_aithne (admin pages + bootstrap-admin enrolment); no impact to lucos_contacts itself |
| **Detected by** | User report (lucas42, via team-lead) |

Source issue: `lucas42/lucos_aithne#106`. Fix: `lucas42/lucos_aithne#107`.

---

## Summary

A deploy of `lucas42/lucos_aithne#105` ("Show contact display names on admin pages") introduced aithne's **first ever outbound HTTPS call** — a `contacts.Get()` to `https://contacts.l42.eu` to resolve a contact's display name. aithne's runtime image is built `FROM scratch` and shipped no CA certificate bundle, so Go's TLS stack had no roots to verify *any* public certificate and rejected the valid Let's Encrypt cert with `x509: certificate signed by unknown authority`. The admin enrol/grants pages — and the bootstrap-admin enrolment flow — could not resolve contact names. It was resolved by copying the CA bundle (and timezone data) into the scratch stage from the Go builder image and redeploying. aithne itself stayed up throughout; impact was confined to the admin contact-name lookups.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 13:11 | `lucas42/lucos_aithne#105` merged and deploy begins — admin pages now call `contacts.Get()` (aithne's first outbound HTTPS) |
| 13:13 | aithne 1.15.10 container live with the new code path |
| 13:17–13:18 | Enrol page hit twice; `contacts.Get("2")` fails — `x509: certificate signed by unknown authority`. Bootstrap-admin enrolment blocked |
| 13:23 | Root cause confirmed (scratch image, no CA bundle); issue `lucas42/lucos_aithne#106` raised with the one-line fix |
| 14:59 | Fix PR `lucas42/lucos_aithne#107` opened (add CA bundle + zoneinfo to the scratch stage) |
| 15:02 | `lucas42/lucos_aithne#107` merged; `#106` closed; deploy pipeline begins |
| 15:06 | aithne 1.15.11 deployed — CA bundle present (150 certs) |
| 15:09 | Verified end-to-end: enrol page resolves the real display name from contacts; no further x509 errors. Incident resolved |

---

## Analysis

### Root cause: scratch runtime image with no CA bundle

aithne compiles to a static Go binary (`CGO_ENABLED=0`) and its runtime stage is `FROM scratch` — only the binary, nothing else. On Linux, Go's TLS client verifies server certificates against the system trust store at `/etc/ssl/certs/ca-certificates.crt`. A scratch image has no such file, so the trust store is empty and the client cannot build a chain to *any* public certificate.

This is not a cert problem on the serving end: `contacts.l42.eu` presented a perfectly valid Let's Encrypt certificate (CN=contacts.l42.eu, issuer R12, notAfter 2026-08-19) that verifies fine from any normally-configured client. The failure was entirely client-side trust.

### Contributing factor: a known-deferred work item that came due silently

The gap was not an oversight in the usual sense — the Dockerfile comment explicitly predicted it:

```dockerfile
# Runtime stage: scratch — the binary is fully static (CGO_ENABLED=0) so there are
# no runtime dependencies. CA certificates and timezone data will be added explicitly
# when the service makes its first outbound HTTPS calls.
FROM scratch
```

The CA-bundle work was deliberately deferred until "the service makes its first outbound HTTPS calls." `lucas42/lucos_aithne#105` was exactly that moment — it added the first outbound HTTPS call — but the deferred work item had no tracking issue and no CI guard, so nothing connected "we are now adding an outbound HTTPS call" to "the deferred CA-bundle task is now due." The latent gap activated silently at deploy time.

**Prevention.** A deferred-work comment in a Dockerfile has no trackable backstop — it relies on whoever later adds the outbound call having read and remembered the comment. A design-time deferral belongs in a tracked issue with a named trigger condition, not in a source comment; the one event that should have made this due (adding the first outbound HTTPS call) sailed straight past because nothing was watching for it. The durable mitigation is a convention rather than a comment: *if a scratch (or distroless-without-certs) image's binary makes any outbound HTTPS call, the final stage must `COPY` `/etc/ssl/certs/ca-certificates.crt` in.* This is captured as a checklist item in the estate-audit follow-up (`lucas42/lucos#240`) so it survives beyond this report. (Per lucos-system-administrator's review.)

Worth naming the second-order judgement too: deferring the CA bundle was itself the wrong call, independent of the tracking failure. The CA bundle plus tzdata is a couple of hundred KB and two `COPY` lines; the saving from omitting them is negligible, while the cost was a latent landmine that detonates silently the first time any outbound call lands. Deferring a near-zero-cost robustness measure to save almost nothing is the anti-pattern. (Per lucos-architect's review.)

### The base-image choice is the deeper root cause

Stepping back from "the CA bundle wasn't copied": the single decision that made this failure possible was choosing `FROM scratch` for the runtime stage. A scratch image ships *nothing* — no CA bundle, no tzdata, no `/etc/passwd`. There is an estate precedent that avoids the entire class: `lucos_docker_health` builds its runtime stage `FROM gcr.io/distroless/static-debian12`, which ships a CA bundle and timezone data by default — you cannot forget to copy something the base already provides.

The honest trade-off: `scratch` has a genuine merit — absolute-minimum attack surface, nothing in the image to exploit — so this is not a blanket "scratch is wrong." But for any service that makes (or might ever make) outbound HTTPS, `distroless/static` is the better default, and aithne is the proof: "makes no outbound calls" was true at creation and silently became false later. This separated into two follow-ups: `lucas42/lucos#240` patches the symptom (audit existing scratch images, encode the CA-bundle convention) and is the agreed action, while `lucas42/lucos#242` carried the class-kill question — "should new Go services default to `distroless/static` unless they provably need nothing outbound?" — as a convention decision for lucas42. They were deliberately split so an open-ended decision wouldn't gate the ready audit work — which is itself the incident's lesson (a tracked issue, not a deferred sub-bullet) applied. **lucas42 decided not to adopt the new default at this stage**, so `#242` is closed and `scratch` remains the default; the `#240` audit + the CA-bundle convention cover the immediate gap. The decision is reversible if the trade-off is worth revisiting later. (Reframe per lucos-architect's review; default decision per lucas42.)

### Why detection was delayed (and impact was contained)

The failure only fires when the outbound-HTTPS code path actually runs, which on these admin pages only happens when someone loads them. `contacts.Get()` is also written as non-fatal — it logs the error and falls back to showing the raw contact ID — so the pages still rendered, just without resolved names. `LUCOS_CONTACTS_ORIGIN` is aithne's only outbound HTTPS dependency, so the blast radius was confined to contact-name resolution. The trade-off: the same non-fatal design that contained the impact also meant a green `/_info` healthcheck never reflected the failure, so it could only be caught by a user hitting the path or by reading the logs. The non-fatal fallback was the *correct* design — the resolution is not to make the call fatal but to *surface* the degraded state (an error counter, or a warning tier in `/_info` distinct from "unhealthy"), a recurring estate theme around distinguishing "degraded" from "unhealthy". Not an action item for this incident.

---

## What Was Tried That Didn't Work

Nothing was tried that failed — diagnosis was first-attempt. Worth recording, though, are two things ruled out early so future responders don't repeat them:

- **It looked like a cert problem on contacts' side.** The initial report was "looks like a cert problem." Checking the served cert (`openssl s_client -connect contacts.l42.eu:443`) showed a valid, unexpired LE cert, which immediately redirected the investigation to the *client* trust store rather than contacts' certificate.
- **A green `/_info` does not prove the fix.** Verification deliberately did **not** rely on the healthcheck. Because the failure is on a non-fatal outbound-only path, `/_info` was green throughout — including while broken. The fix was confirmed by exercising the real path end-to-end (generating a bootstrap enrol invite and GETting `/enrol`, which resolved the real display name "Luke Blaney" from contacts) and confirming zero x509 errors in the logs after the redeploy.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Add CA bundle + zoneinfo to aithne's scratch stage | `lucas42/lucos_aithne#107` | Done (merged 15:02 UTC, deployed 15:06 UTC) |
| Audit other `FROM scratch` Go services for the same latent gap, and encode the "scratch + outbound HTTPS ⇒ COPY the CA bundle" convention | `lucas42/lucos#240` | Open |
| Decide whether new Go services should default to `distroless/static` instead of `scratch` (convention default) | `lucas42/lucos#242` | Closed — lucas42 declined the new default at this stage; `scratch` remains the default, `lucas42/lucos#240` covers the gap. Reversible if revisited |
| Surface a degradation signal when the contacts name lookup falls back to the raw contact ID (pre-existing UX concern surfaced by this incident) | `lucas42/lucos_aithne#108` | Open |
| Add an admin invite-revocation endpoint (the store has no revocation path; surfaced when the verification test invite could not be voided) | `lucas42/lucos_aithne#109` | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

To exercise the failing path end-to-end during verification, a single-use bootstrap enrolment invite was generated for the bootstrap-admin contact. The GET test does not consume the invite (it is consumed only at `POST /enrol/finish`), so the invite remained valid afterwards. The invite token is sensitive (it grants passkey setup for the bootstrap-admin) and has been kept out of this report and all messages, and the local copies were shredded. team-lead was notified that the unconsumed invite exists.

The invite store (`store/enrolment_invite.go`) has no revocation path and no single-active-invite enforcement: `CreateInvite` is a plain insert, so a fresh `--bootstrap-invite` creates a *new* valid invite **alongside** the old one — both remain valid independently until each expires or is used (it does not supersede or revoke the existing one). The only routes to closure are consumption at `POST /enrol/finish`, natural expiry (`InviteTTL = 24h`, so ~24h after creation), or a direct DB update of `used_at`. **Security review (lucos-security) assessed the residual as acceptable to let expire naturally**: the raw token never entered any public channel, this report, or any message; POST response bodies are not typically captured in access logs; the TTL window is short; and the bootstrap-admin has no pre-existing passkey, so the token's worst case is a session-creation event rather than escalation on an active account. The missing revocation capability is tracked as a follow-up (see Follow-up Actions).
