# Incident: lucos_mail down — alpine base bump pulled in a breaking Dovecot 2.4

| Field | Value |
|---|---|
| **Date** | 2026-06-11 |
| **Duration** | ~14 minutes (11:42 → ~11:56 UTC) of smtp down |
| **Severity** | Outage — estate mail delivery (SMTP) down |
| **Services affected** | lucos_mail (`lucos_mail_smtp` / postfix + dovecot on avalon). `lucos_mail_docs` unaffected. |
| **Detected by** | Reported via team-lead — red `main` pipeline noticed during the estate Dependabot rollout (not by monitoring; see Detection gap) |

---

## Summary

A Dependabot PR (lucas42/lucos_mail#58) bumped the postfix container's base image `alpine:3.21.1 → alpine:3.24.0` and auto-merged. Alpine 3.22+ ships **Dovecot 2.4**, which introduced a breaking config-format change: `dovecot.conf` must begin with a `dovecot_config_version` setting. The repo's `postfix/dovecot.conf` predates that requirement, so Dovecot failed fatally on every start (`doveconf: Fatal: … line 2: The first setting must be dovecot_config_version`). The `lucos_mail_smtp` container crash-looped, postfix never opened port 25, and the deploy healthcheck (`nc -z 127.0.0.1 25`) failed.

The container **built** cleanly — the breakage manifests only at runtime/deploy, which is why it passed the auto-merge CI gate. Service was restored by rerunning the last-green pipeline (redeploying the working alpine-3.21.1 image), then the base image was pinned back to `alpine:3.21.1` (PR lucas42/lucos_mail#59) to make `main` green durably. Re-applying the alpine upgrade requires a Dovecot 2.4 config migration, tracked in `lucas42/lucos_mail#60`.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 11:15:55 | Dependabot lucas42/lucos_mail#58 (`alpine:3.21.1 → 3.24.0`) auto-merges → pipeline 151 |
| 11:42:04 | First deploy of the broken `1.0.20` image to avalon — smtp container recreated, dovecot fails fatally, crash-loop begins, **mail down** |
| 11:42 → 11:50 | Repeated deploy attempts fail (healthcheck on port 25); rollout's CI-green check over-retries the failed workflow (~14 queued reruns) |
| ~11:50 | SRE engaged via team-lead |
| 11:51 | Cancelled the ~14 queued/running reruns of the broken pipeline (stop redeploying the broken image); verified the avalon deploy lock was **not** orphaned (other avalon deploys still landing) |
| 11:51 | Reran the last-green pipeline (pipeline 146) to restore service |
| 11:54:45 | Pin PR lucas42/lucos_mail#59 (`alpine → 3.21.1`) approved by lucas42 and auto-merged → pipeline 156 |
| ~11:56:30 | Restore deploy completes — `lucos_mail_smtp` Up (healthy), port 25 listening, **mail restored** |
| ~11:57 | Pipeline 156 green → `main` green |

---

## Analysis

### Root cause: a major base-image bump carried a breaking runtime dependency

`postfix/Dockerfile` uses a bare `FROM alpine:3.21.1` and installs `postfix dovecot` via `apk`. Bumping the base to `alpine:3.24.0` moved the apk-provided Dovecot from 2.3.x to **2.4.x**. Dovecot 2.4 made `dovecot_config_version` a mandatory first setting in `dovecot.conf` (part of a wider config-format change that also renamed/removed settings). The repo's config opens with a comment then a `service auth { … }` block — no version line — so `doveconf` aborts. The container's `startup.sh` depends on dovecot, so the whole container fails its healthcheck and crash-loops.

This is a base-image upgrade carrying a **breaking change in a bundled package**, not a Dockerfile or application bug. The bump itself is desirable (security currency); it simply can't be taken until the config is migrated.

### Why it passed auto-merge: no test exercises the running stack

`apk add postfix dovecot` succeeds at **build** time — dovecot's config is only parsed when the daemon **starts**, which happens at deploy/runtime. lucos_mail has **no CI test job** that starts the stack, so nothing between merge and production ever ran `doveconf` or brought postfix up. The Dependabot auto-merge gate had only a green build to look at, so it merged on a green build while the latent runtime failure waited for the deploy step. Every subsequent deploy attempt then failed identically — a persistent failure, not a flaky one. (The ~14 queued workflows on the failing pipeline were retries from the rollout's CI-green check, not independent failures.)

The sharp lesson (lucas42's framing): the gap isn't "auto-merge should gate on a successful *deploy*" — it's that **a test asserting the mail stack actually comes up would have failed in CI and blocked the auto-merge outright**, no deploy needed. That test is the durable fix, tracked in `lucas42/lucos_mail#61`.

### Detection gap: monitoring doesn't probe the mail path

`lucos_mail`'s monitored checks are `tls-certificate`, `fetch-info`, `circleci`, and `apache` — all of which exercise the **docs/web** side (`lucos_mail_docs`), which stayed healthy throughout. **Nothing probes SMTP / port 25 / actual mail delivery.** During the outage, monitoring's only mail-related signal was the `circleci` check going red (a *CI-failure* signal, which is why this surfaced as "broken main pipeline" rather than "mail is down"). A complete smtp outage was otherwise invisible to monitoring. Had this not coincided with a watched rollout, the red CI check is the only thing that would have flagged it — and that signals "a pipeline failed", not "mail is not being delivered".

---

## What Went Well

- **Restore was clean and fast.** Rerunning the last-green pipeline redeployed a known-good image through the normal deploy path (correct volumes, no hand-rolled container on a mail server). ~14 min total outage.
- **The deploy-lock concern was checked, not assumed.** Cancelling ~14 in-flight workflows raised the risk of orphaning the estate-wide avalon deploy lock; this was verified against live Loganne deploy events (other avalon deploys kept landing) before concluding it was safe — avoiding a false "estate deploy freeze" diagnosis.

## What Was Tried / Notes For Next Time

- **Don't hand-roll the mail container to shave minutes.** Production hosts have no persistent compose directory (compose deploys transiently during CI), and `lucos_mail_smtp` carries mail-data volumes — a `docker run` reconstruction risks misplacing mail. The CI redeploy of the last-green build was the correct restore even though it costs a build cycle.
- **Diagnose at the right layer.** A red CI pipeline on a base-image bump is a *runtime* failure until proven otherwise; the `lucos/build` job succeeding ruled out the build immediately and pointed straight at the deploy/healthcheck.

---

## Follow-ups

- **`lucas42/lucos_mail#61` — add a CI test job that fails when the mail stack doesn't start (the durable fix).** This is the real remediation: a test that brings the stack up would have caught the Dovecot 2.4 breakage in CI and **blocked the auto-merge**, rather than letting it ship and fail at deploy. It also closes the recurrence risk — once it's in, the alpine 3.24 bump fails CI and can't auto-merge, so no ignore/hold is needed.
- **`lucas42/lucos_mail#60` — Dovecot 2.4 config migration.** Migrate `dovecot.conf` to 2.4 syntax so the alpine upgrade can be re-applied (then the base can move past 3.21). Until either #61 (blocks the bad bump) or #60 (makes the bump safe) lands, Dependabot's daily docker updater would otherwise re-open and auto-merge the alpine 3.24 bump and re-break mail — covered in the interim by team-lead owning the next-run deadline.
- **Monitoring coverage: no smtp/mail-delivery probe on lucos_mail.** A synthetic "is port 25 accepting / can we send a test mail" check would have surfaced this as "mail down" rather than "CI red". Worth weighing against the cost of a synthetic-mail check (test-message hygiene, per-host config) before building — captured here as a known gap rather than an automatic action.
