# Incident: lucos_mail down after a malformed DOVECOT_USERS credential

| Field | Value |
|---|---|
| **Date** | 2026-10-01 |
| **Duration** | ~4–5 minutes (between 22:51:24 and ~22:52:10 UTC, to 22:56:25 UTC) |
| **Severity** | Complete outage |
| **Services affected** | lucos_mail (SMTP on port 25: inbound MX for l42.eu, and the authenticated relay used by lucos_monitoring and the NAS) |
| **Detected by** | lucos-site-reliability's log watch on `lucos_mail_smtp`, set up for the post-deploy check of lucas42/lucos_mail#84. lucos_monitoring did not alert. |

---

## Summary

lucas42/lucos_mail#84 changed lucos_mail so that its SASL users file is written at startup from the `DOVECOT_USERS` credential, and so that the container refuses to start if that value is malformed. About 25 minutes after it deployed, a third account (`campaigns@l42.eu`, for lucas42/lucos_campaigns#19) was added to the production credential. Its new line had a stray `#` after every `$` in the crypt hash.

The next lucos_mail deploy replaced the working container with one that failed the startup check on every attempt. Port 25 stopped answering for about four to five minutes, until lucas42 corrected the credential and redeployed.

The fail-closed check did exactly what it was designed to do. The outage came from it guarding *every* account, so one bad new line took down the two existing, valid ones with it.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 22:10:52 | lucos-code-reviewer approves lucas42/lucos_mail#84, noting that the production `DOVECOT_USERS` must have every `$` written as `$$`, which it can't verify. |
| 22:22:48 | lucas42 approves lucas42/lucos_mail#84. |
| 22:23:02 | lucas42/lucos_mail#84 merged: SASL users are now sourced from lucos_creds `DOVECOT_USERS`, and the container fails closed if the value is malformed. |
| 22:24:35 | `lucos_mail_smtp` v1.0.35 starts with two valid users, `monitoring@l42.eu` and `nas@l42.eu`. Post-deploy verification and an SRE log watch begin. |
| 22:37:57 | lucas42/lucos_campaigns#58 merged: lucos_campaigns can send through lucos_mail. |
| 22:48:01 – 22:49:33 | Production credentials updated: `MAIL_PASSWORD` (twice) and `MAIL_DRIVER` in lucos_campaigns, and `DOVECOT_USERS` in lucos_mail at 22:49:15. The new `DOVECOT_USERS` has three lines. The `campaigns@l42.eu` line is malformed. |
| 22:50:31 | lucos_mail CircleCI pipeline 202 is triggered via the API to pick up the new credential. |
| 22:51:24 | Pipeline 202's `lucos/deploy-avalon` job starts. At some point between here and about 22:52:10, the healthy v1.0.35 container is replaced by v1.0.36, which exits on startup: `DOVECOT_USERS is malformed: …`. **Outage begins.** |
| 22:52:19 | The SRE log watch reports the container restart and six `DOVECOT_USERS is malformed` lines (restart count 6). |
| 22:52:32 | SRE confirms the container is crash-looping (`status=restarting`, 8 restarts) and that the only change was the credential. |
| ~22:53 | SRE runs the image's own startup checks against the live value in a throwaway container. Checks 1 and 2 pass; check 3 (CRYPT form) fails on the `campaigns@` line only. Root cause and fix are sent to team-lead, and a terminal notification goes to lucas42. |
| 22:53:00 | `lucos/deploy-avalon` (job 676) fails because the container never becomes healthy. The old container has already been replaced. |
| 22:54:45 | lucas42 updates production `DOVECOT_USERS` with a corrected `campaigns@` line. |
| 22:55:02 | lucos_mail pipeline 204 is triggered. |
| 22:56:23 | `lucos_mail_smtp` v1.0.37 starts. |
| 22:56:25 | Postfix `daemon started`. **Service restored.** team-lead independently sees the Postfix banner on port 25 at 22:56:25. |
| 22:56:32 | Loganne: `Deployed lucos_mail v1.0.37 to avalon.s.l42.eu`. |
| 22:56:37 | SRE confirms 0 restarts, healthy, and three users in `/etc/dovecot/users`, all `{SHA512-CRYPT}$6$…` with no `#`. |

The exact moment the outage began is not known. Docker's event buffer on avalon no longer held the container events for that window. The start bound is the deploy job's start time. The end bound is estimated from the restart count at 22:52:19 and Docker's restart backoff.

---

## Analysis

### Root cause: a malformed line in a new credential value

Confirmed by reproduction. The image's startup validation (`postfix/config/startup.sh`) was run against the live `DOVECOT_USERS` value inside a throwaway, network-less `lucas42/lucos_mail_smtp:1.0.36` container, printing only line structure:

- The value had three lines, all `{SHA512-CRYPT}`.
- Check 1 (at least one well-formed `<address>:{scheme}hash` line) passed.
- Check 2 (no line that isn't well-formed) passed.
- Check 3 (every `CRYPT` hash in `$id$salt$hash` form) **failed, on the `campaigns@l42.eu` line only**.
- In that line, each of the three `$` separators was followed by a `#`, so the crypt id read `#6` instead of `6`. Removing the three `#` gave field lengths identical to the two valid lines: 16-character salt, 86-character hash. The hash itself looks intact.

**How the `#` characters got there: a slip during manual escaping.** The credential is entered by hand, and every `$` has to be written as `$$`, because Compose would otherwise interpolate it (see the comment in `startup.sh` and the lucos_mail README). lucas42 explained in chat (no written artefact) that while adding the extra `$` characters by hand, he typed `#`, which is next to `$` on his keyboard, three times. So the root cause is the manual escaping step itself: a human hand-editing a 100-character secret, with no preview and no feedback until the next deploy. Two untested hypotheses were raised in review before this was known: that the `$$` instructions themselves produced the `#` (lucos-developer doubted it), and that `.env` comment handling played a part (lucos-system-administrator). Neither was the cause.

### Contributing factor: one bad line rejects every account

The check validates the users file as a whole. Any malformed line, including one for a brand-new account nobody yet depended on, stops the container. That took down inbound mail and the two existing, valid relay accounts with it.

The fail-closed design was deliberate: the comment in `startup.sh` says it exists to *"reject them rather than start with logins that can never succeed"*. It worked as designed. The trade-off it implies is that the *availability* of the whole service depends on every line in a hand-edited credential being right.

lucos-architect's review argues this escalation isn't actually *required* by fail-closed. A malformed hash can't authenticate anyone, so dropping only its line is still fail-closed for that account. Rejecting the whole file instead turned a one-account fault into a service-wide outage, including inbound MX, which doesn't use SASL at all. Reviewers disagree on what should happen when *no* line is valid. That question, and the whole-file versus per-line choice, are with lucas42 in lucas42/lucos_mail#85.

### Contributing factor: the error didn't say what was wrong

The startup error (`DOVECOT_USERS is malformed: want one <address>:{scheme}hash per line …`) is the same message for all three checks. It names neither the failing line nor the rule. Finding that it was check 3 on the `campaigns@` line took a reproduction in a throwaway container. Printing the line number, the address and the failed rule would have given the cause straight away, without exposing any hash. (Raised by lucos-ux.)

### Contributing factor: a deploy replaces a working container before the new one proves healthy

The `lucos/deploy-avalon` job replaced the running v1.0.35 container, then failed when v1.0.36 never became healthy. There is no rollback, so the failed deploy left the service down until a fixed deploy followed. This is a known, accepted property of estate deploys, not something specific to lucos_mail. lucas42/lucos_deploy_orb#192 ("A failed deploy leaves the service DOWN, with no working rollback") is closed as not planned. lucas42 closed the auto-rollback implementation, lucas42/lucos_deploy_orb#194, unmerged: *"No. This is adds much too complexity to handle a relatively rare occurrence."* No rollback follow-up is proposed here.

What this incident adds is that a credential edit is **deferred damage**. lucos_creds does no validation on write, and lucos_mail's CI never sees production credentials. So the first place a malformed `DOVECOT_USERS` gets checked is inside the deploy that replaces the working container. Here the deploy was triggered by the edit itself, so the two were minutes apart. They could equally have been days apart. (Point raised by lucos-system-administrator in review.)

lucos-architect frames this as the check running at the latest possible moment, after the old container is gone. The same check at credential-write time, or as a pre-deploy step, would have turned this outage into a refused change. The editing step that introduced the fault has its own weakness. Writing every `$` as `$$` is a manual step on a 100-character secret, with no preview, and the only feedback is a failed deploy later (lucos-ux, lucos-architect).

### Detection: no monitoring alert

lucos_monitoring has a `port-25-reachable` check for lucos_mail, but loganne has no `monitoringAlert` for lucos_mail during the outage. **Why is unverified.** The likeliest explanation is that the deploy windows of pipelines 202 and 204 suppressed alerting for nearly the whole outage. The outage started inside the first deploy window, and the gap between them was 22:53:00 to 22:55:02, about two minutes. The outage was found only because an SRE log watch happened to be running for the lucas42/lucos_mail#84 post-deploy check. If deploy-window suppression is confirmed, a failed deploy arguably should end suppression or raise its own alert (lucos-architect). That stays conditional until the hypothesis is checked.

---

## What Was Tried That Didn't Work

Nothing was attempted that failed. A container restart was deliberately **not** tried: with `restart: always`, the container was already restarting continuously with the bad value baked into its environment, so only a corrected credential plus a redeploy could help.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Decide whether a malformed `DOVECOT_USERS` line should take down only that account rather than the whole service. If approved, it must come with lucos-security's and lucos-developer's conditions: write only validated lines, stay fail-closed when no line is valid, never echo a skipped line, and make skips visible to monitoring. The ticket also covers the alternatives: validating a value before saving it, one credential per account, or a `$`-free hash scheme. | lucas42/lucos_mail#85 | Awaiting decision |
| Remove the manual `$$` escaping step, the confirmed root cause, rather than documenting it better. lucas42 has also proposed storing plaintext passwords in lucos_creds instead of hashes, so that nothing needs escaping. lucos-architect and lucos-security are assessing that; TBD pending their outcome. | lucas42/lucos_creds#484 (ADR-0006); TBD | Under assessment |
| Validate the value before it can do damage, either at credential-write time, as a pre-deploy step, or with a standalone check plus an escaping command in the README. Also name the failing line and rule in the startup error. These are tracked as options alongside the per-line decision. | lucas42/lucos_mail#85 | Awaiting decision |

Already tracked elsewhere and not caused by this incident: AUTH offered before STARTTLS on port 25 (lucas42/lucos_mail#83); Dovecot's own auth logs being dropped inside the container, which was noted during the post-deploy check and deliberately not ticketed unless a mismatch occurs.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

The incident concerns a credential (`DOVECOT_USERS`, SMTP password hashes). This report describes only its structure: line count, account names, hash scheme and field lengths. No hash or salt values are included.

During recovery verification, a faulty `sed` in the SRE's structure-only check printed the three full SHA512-crypt hashes into the SRE agent's **local** tool output. They were not posted to GitHub, sent to any teammate, or written to disk, but they are present in that session's transcript.

lucos-security's assessment: no emergency action is needed. Fold the three accounts into the next rotation, unless any underlying password is human-chosen, in which case rotate that one now. Whether to rotate `campaigns@l42.eu` early is lucas42's call, since rotating means re-entering both sides by hand. The SRE persona now requires secret structure checks to print derived lengths only, never values, in `agents/lucos-site-reliability.md` and `references/agent-github-identity.md` in lucas42/lucos_claude_config. lucos-security also checked this report's diff and found no hash or salt material in it.
