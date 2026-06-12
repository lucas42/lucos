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

**Prevention.** A deferred-work comment in a Dockerfile has no trackable backstop — it relies on whoever later adds the outbound call having read and remembered the comment. The durable mitigation is a convention rather than a comment: *if a scratch (or distroless-without-certs) image's binary makes any outbound HTTPS call, the final stage must `COPY` `/etc/ssl/certs/ca-certificates.crt` in.* This is captured as a checklist item in the estate-audit follow-up (`lucas42/lucos#240`) so it survives beyond this report. (Per lucos-system-administrator's review.)

### Why detection was delayed (and impact was contained)

The failure only fires when the outbound-HTTPS code path actually runs, which on these admin pages only happens when someone loads them. `contacts.Get()` is also written as non-fatal — it logs the error and falls back to showing the raw contact ID — so the pages still rendered, just without resolved names. `LUCOS_CONTACTS_ORIGIN` is aithne's only outbound HTTPS dependency, so the blast radius was confined to contact-name resolution. The trade-off: the same non-fatal design that contained the impact also meant a green `/_info` healthcheck never reflected the failure, so it could only be caught by a user hitting the path or by reading the logs.

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
| Audit other `FROM scratch` Go services across the estate for the same latent gap (no CA bundle + a current or future outbound HTTPS call) | `lucas42/lucos#240` | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

To exercise the failing path end-to-end during verification, a single-use bootstrap enrolment invite was generated for the bootstrap-admin contact. The GET test does not consume the invite (it is consumed only at `POST /enrol/finish`), so the invite remained valid afterwards. The invite token is sensitive (it grants passkey setup for the bootstrap-admin) and has been kept out of this report and all messages, and the local copies were shredded. A fresh `--bootstrap-invite` supersedes it for the actual bootstrap. team-lead was notified that the unconsumed invite exists.
