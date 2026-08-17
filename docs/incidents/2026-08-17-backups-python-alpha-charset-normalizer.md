# Incident: lucos_backups crash-looped for 15h — CPython alpha base image met a wheel it couldn't load

| Field | Value |
|---|---|
| **Date** | 2026-08-17 |
| **Duration** | ~15h24m — container recreated broken ≈07:21 UTC, service restored 22:45:07 UTC. Independently, the monitoring alert window ran 07:32:00 → 22:46:13 UTC (15h14m13s) |
| **Severity** | Complete outage of the backup service — data risk |
| **Services affected** | `lucos_backups` on avalon (the single container backing up avalon, xwing, salvare and aurora). `lucos_docker_health` red as a consequence. |
| **Detected by** | Monitoring alerted correctly at 07:32 UTC. **Acted on** 15 hours later, by SRE + sysadmin ops checks at ~22:26 UTC. |

Originating issue: lucas42/lucos_backups#390. Fix: lucas42/lucos_backups#391.

---

## Summary

On 2026-08-06 Dependabot bumped `lucos_backups`' base image from `python:3.14.6-alpine` to `python:3.15.0a2-alpine` — a CPython **alpha** — classified the change as `version-update:semver-minor`, and auto-merged it. Nothing broke, and the container ran normally for 11 days.

On 2026-08-17 the trap sprang. `charset_normalizer` 3.5.1 had been published to PyPI, carrying a `cp315` binary wheel compiled against a *later* 3.15 pre-release whose C-API type-slot table is wider than alpha 2's. The next build picked it up and produced an image that dies on import with `SystemError: module charset_normalizer.cd uses unknown slot ID 84`. The failing deploy had already recreated the container, so `lucos_backups` was left crash-looping — `RestartCount` was 877 when I first inspected it at ~22:27 and 878 three minutes later when lucas42/lucos_backups#390 was filed, which is itself a fair measure of the ~4.5-second cycle.

For 15 hours and 24 minutes no host was backed up, and the 15:25 UTC `create-backups` run did not happen. Resolved by reverting the base image to `python:3.14.6-alpine` (lucas42/lucos_backups#391).

**On the data impact, stated precisely, because the headline number overstates it.** `create-backups` carries a 20-hour skip-if-fresh threshold, so the twice-daily cron (03:25 and 15:25) is effectively *once* daily with the second slot as a safety net. The 03:25 run completed successfully at 03:26, before the outage began, and the next ran at 03:25 the following morning — so the interval between completed backups stayed within the normal ~24h cadence. **The data risk was real but not realised**; had the outage continued another 4.5 hours, a genuine daily cycle would have been lost. What *was* lost is the safety-net run, and it was a real loss rather than a no-op: the marker that would have caused it to skip lives in `/var/run` inside the container (no volume is mounted), so the 07:21 recreate destroyed it and a healthy 15:25 run would have performed a full backup.

Two things about this incident are worth more attention than the bug itself. The build **does not honour `Pipfile.lock`**, which is why a commit that touched only a GitHub Actions workflow file was sufficient to detonate it. And monitoring did its job — it alerted 11 minutes after the failure — yet the outage still ran 15 hours, because detection and response are not the same thing.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| **2026-08-06** 07:06 | Dependabot opens the `minor-and-patch` docker group bump: `python` `3.14.6-alpine` → `3.15.0a2-alpine`, classified `version-update:semver-minor`. Auto-merged (commit `db6fe3c`). **Latent from here.** |
| 2026-08-06 → 08-11 | Several builds and deploys. All healthy — no wheel yet exists that the alpha cannot load. |
| **2026-08-11** 07:16 | Image `1.4.31` built. Ships `charset_normalizer` 3.4.9. Runs healthily for six days. |
| 08-11 → 08-17 | `charset_normalizer` 3.5.1 published to PyPI. Exact publication time not established; it is bounded by these two builds. |
| **2026-08-17** 07:03:00 | Last successful `config` scheduled job run. |
| 07:07:14 | Last successful `tracking` scheduled job run. **Service still fully healthy.** |
| 07:11:42 | Pipeline 816 starts — PR lucas42/lucos_backups#388, a `github/codeql-action` bump. **Changes no Python dependency.** |
| 07:14:38 | Image `1.4.32` built. Because the build re-resolves, it picks up `charset_normalizer` **3.5.1** — while the `Pipfile.lock` inside that same image still pins `3.4.9`. |
| 07:18:35 | Pipeline 819 starts — PR lucas42/lucos_backups#389, the `charset-normalizer` lockfile bump. |
| **≈07:21** | Pipeline 816's `deploy-avalon` runs `docker compose up`; the container is recreated on `1.4.32` and immediately begins crash-looping. **Service down from here.** (Bracketed by the job's 07:21:02 start and 07:23:04 failure.) |
| 07:21:48 | Image `1.4.33` built — same defect. |
| 07:23:04 | Pipeline 816 `deploy-avalon` **fails**: `container lucos_backups is unhealthy`. The container is left in place, broken. |
| 07:32:00 | **First monitoring alert** — `lucos_backups` `fetch-info`. Email delivered and accepted by Gmail. |
| 07:37:02 → 07:38:42 | Pipeline 819 `deploy-avalon` fails identically, having replaced the container again with `1.4.33`. |
| 07:42:00 | `lucos_docker_health` avalon check begins failing: `Unhealthy containers: lucos_backups`. |
| 10:08:02 | `lucos_backups` now showing 4 failing checks: `circleci`, `config`, `fetch-info`, `tracking`. |
| 13:57 → 15:38 | **Unrelated:** a CircleCI API outage trips 236 alerts across 54 systems. `lucos_backups`' own re-alerts at 14:19, 14:47, 15:01 and 15:11 land inside that window. Tracked separately as lucas42/lucos_monitoring#302. |
| 15:25 | Scheduled `create-backups` run **does not happen** — the safety-net slot, lost. (Not a no-op: the 07:21 recreate had wiped the in-container skip-if-fresh marker, so a healthy run here would have done a full backup. See Summary.) |
| ~22:26 | SRE ops check reads the monitoring API and finds `lucos_backups` failing; container is `Restarting`, `RestartCount=877`. |
| 22:30:25 | `lucos-system-administrator` files lucas42/lucos_backups#390 from its own ops check. |
| 22:35:37 | SRE attempts service restoration by re-running the last-good pipeline's deploy job on CircleCI. **This does not roll back** — see "What Was Tried That Didn't Work". |
| 22:42:13 | lucas42/lucos_backups#391 approved by `lucos-code-reviewer`. |
| 22:42:44 | Approved by lucas42; merged 22:42:56. |
| 22:45:07 | Image `1.4.34` deployed, container healthy. **Service restored.** |
| 22:46:13 | `monitoringRecovery` — all checks healthy. |
| ~22:52 | Ad-hoc `refresh-config` and `refresh-tracking` runs confirmed successful against schedule-tracker (ages 21s and 7s, 0 errors). |
| ~23:07 | Ad-hoc `create-backups` run executed end-to-end on the new interpreter — real archives created and distributed to aurora, salvare and xwing. Killed two-thirds through by the invoking command's own 540s timeout; left no partial state. |
| ~23:20 | `lucos_repos` `stale-dependabot-prs` clears on the next audit sweep. Estate returns to **55/55 healthy, 0 failing, 0 unknown**. |

---

## Analysis

### Stage 1 — a CPython alpha entered the estate as a "minor" bump

Dependabot's version parser reads `3.14.6-alpine` → `3.15.0a2-alpine` as a minor version increase and labels it `version-update:semver-minor`. Under the `minor-and-patch` group that classification carries auto-merge. No human, and no CI gate in this repo, ever saw the word "alpha".

This is the fourth production break of this class (see lucas42/lucos#273, which was opened after the third and is still Awaiting Decision). The distinguishing feature of this one is **latency**: the previous three broke on the deploy that introduced them. This sat harmless for 11 days, because the alpha interpreter was perfectly capable of running every wheel that existed on 2026-08-06.

### Stage 2 — a third party published the wheel that completed the trap

`charset_normalizer` ships compiled C extensions. Version 3.5.1's `cp315` wheel was built against a later 3.15 pre-release, and declares a type slot (ID 84) that CPython 3.15.0a2 — tagged `main, Dec 4 2025` — does not know about. Importing it raises `SystemError: module charset_normalizer.cd uses unknown slot ID 84`.

Reproduced directly on avalon, with both controls:

```
python:3.15.0a2-alpine + charset_normalizer==3.5.1  →  SystemError: unknown slot ID 84
python:3.14.6-alpine   + charset_normalizer==3.5.1  →  IMPORT OK 3.5.1
```

`charset_normalizer` is not a declared dependency of this project. It arrives transitively via `requests`, and is imported during `requests`' own module initialisation — so it takes down anything that imports `requests`, which here is the server's very first import.

Worth recording for the estate: **`rc` is materially safer than `a`/`b`.** CPython freezes its ABI at release-candidate stage. `python:3.15.0rc1-alpine` + `charset_normalizer==3.5.1` imports cleanly — tested, because two other repos are on exactly that tag.

### Stage 3 — the build ignores `Pipfile.lock`, so an unrelated commit shipped the wheel

`RUN pipenv install` re-resolves dependencies at build time rather than installing what the lockfile pins. This is not inference; it is visible inside the artefacts:

| image | `Pipfile.lock` *inside the image* pins | actually installed |
|---|---|---|
| 1.4.32 | `charset-normalizer==3.4.9` | **3.5.1** |
| 1.4.33 | `charset-normalizer==3.5.0` | **3.5.1** |
| 1.4.34 (the fix) | `charset-normalizer==3.5.0` | **3.5.1** |

Three consequences, in ascending order of importance:

1. **A workflow-file edit became an outage.** Pipeline 816 bumped `github/codeql-action` and nothing else. It still rebuilt, still re-resolved, and still shipped the poisoned image. Its deploy failed at 07:23:04 — *before* the `charset-normalizer` lockfile PR had deployed at all.
2. **The obvious remediation was a no-op.** lucas42/lucos_backups#390's original diagnosis, reasoning from the commit history, attributed the break to the `charset-normalizer` bump in #389 and proposed reverting or pinning it. That was a reasonable read of the evidence available and it was wrong: pinning a file the build does not read changes nothing.
3. **The repo cannot tell you what is deployed.** Every Dependabot lockfile PR here is decorative. Filed as lucas42/lucos_backups#392; note the fix shipped today does *not* address this — production is still running 3.5.1 against a lock that says 3.5.0.

`lucos-security`'s framing of this is stronger than the reproducibility one and is worth stating in its own right: because the build re-resolves on every commit, there is **no code-review gate at all** between a package publishing on PyPI and that code executing on a host holding production SSH credentials. Dependabot exists to provide precisely that gate, and it is being silently routed around. This is not specific to `lucos_backups` — an estate sweep found **8 of 8** Python repos installing without a lockfile-enforcing flag (lucas42/lucos_repos#488), though the *consequence* is confirmed by artefact inspection only for `lucos_backups`.

### Stage 4 — a failed deploy leaves the service broken, not unchanged

`docker compose up` recreates the container and *then* waits on the healthcheck. When the healthcheck never passes, the deploy step reports failure — but the previous, working container is already gone. A red CI pipeline therefore does not mean "production is untouched"; here it meant "production is destroyed, and we told you by failing a build".

Both of the day's deploys failed this way, and the CI failure was visible in monitoring (`circleci` check red) from 07:48. That signal was correct and available for 15 hours.

### Stage 5 — the observer reported 42 recoveries during an outage that never recovered

`lucos_docker_health` watches avalon's containers and correctly went red at 07:42. It then flapped for the rest of the day. Over the identical 15-hour condition:

| system | `monitoringAlert` | `monitoringRecovery` |
|---|---|---|
| `lucos_backups` (the broken service) | 7 | **0** |
| `lucos_docker_health` (the observer) | 51 | **42** |

Same fault, same window. The service's own checks never once claimed recovery — correctly, because it never once recovered. The observer announced recovery 42 times.

The mechanism is in `lucos_docker_health/main.go`, which flags a container in exactly two cases: health status `unhealthy`, or status `starting` where `time.Since(startedAt) > stuckStartingThreshold` (5 minutes). A container restarting every ~4.5 seconds never has a `StartedAt` older than ~4.5 seconds, so **the stuck-starting guard — which exists precisely to catch a container that never comes up — cannot fire during a restart loop.** It measures how long *this incarnation* has been starting, when the meaningful quantity is how long the container has been failing to reach healthy; the loop resets that clock faster than the guard's threshold. That leaves only the `unhealthy` branch, and with `interval: 10s, retries: 3, start_period: 30s` reaching `unhealthy` needs ~30s of continuous uptime that a 4.5s loop rarely provides. The container's health log holds five entries for the whole fifteen hours.

The check's own debug strings confirm individual runs kept succeeding — `Last 23 runs of scheduled job errored`, then `Last 19`, `Last 9`, `Last 5`, the counter resetting each time. Compounding it, the check asks whether **any** of the 5 most recent runs succeeded, so one lucky sample in five holds it green: four correct observations of a dead service are overridden by one. That tolerance is right for a nightly job and inverts the meaning for a 60-second sampling probe.

**A crash-loop is one outage, not many.** `lucos_backups` died during module import, before binding its port; it served zero requests for 15h24m and there was no instant at which it worked. False recoveries are worse than noise, because a reader watching that feed sees something repeatedly fixing itself — which is an argument not to investigate. Some of the delay in Stage 6 may well be exactly that.

### Stage 6 — detection worked; response didn't

This is the part that cost the 15 hours, and it deserves to be stated plainly rather than folded into a lesson.

Monitoring behaved *well*. It alerted 11 minutes after the container broke. It escalated to four failing checks. It correctly propagated to `lucos_docker_health`. Alert emails were generated and accepted by Gmail — `lucos_mail_smtp` shows `status=sent` with `250 2.0.0 OK` for every one, and zero deferrals or bounces all day. Nothing was suppressed, nothing was lost, and no threshold was mistuned.

The outage lasted 15 hours anyway, because nothing in the system converts a correct alert into someone acting on it. It was found by the next scheduled ops check.

The previous report in this series (2026-07-31) concluded that "the quiet failures are the expensive ones" — that one lasted hours because it failed *politely* while `/_info` stayed green. Today's failure was as loud as an outage can be: crash-looping container, four red checks, two red pipelines, a red dependent system, and delivered email. **It lasted more than twice as long.** Loudness is not the variable that matters; whether anyone is listening is.

---

## What Was Tried That Didn't Work

**Rolling back via a CircleCI job re-run.** With the deploy driven from CircleCI over `DOCKER_HOST=ssh` there is no compose project directory on avalon, so `docker compose` cannot be driven from the host. The apparent minimal-intervention restore was to re-run the `deploy-avalon` job from pipeline 813 — the last pipeline whose deploy succeeded, which built the healthy `1.4.31`.

It redeployed the **broken 1.4.33**. The orb's "Determine deployed version from git tags" step resolves the *newest* tag rather than the tag at the pipeline's own commit, so a re-run of an old pipeline deploys current `main`'s version. The job log says `Deploying version: 1.4.33` in plain text.

This is worth knowing before the next incident: **on this estate, re-running an old pipeline is not a rollback.** It cost ~7 minutes here. Fix-forward was the only available path.

**A probe that used the wrong module name.** While verifying the new interpreter, `importlib.import_module("scripts.create_backups")` reported `ModuleNotFoundError`. The file is `create-backups` (hyphenated, invoked as `python -m scripts.create-backups`); the module is fine. Noted only because a `ModuleNotFoundError` in a post-incident verification log is exactly the sort of thing that reads as a second bug.

---

## Verification

- Container healthy on `1.4.34`, `RestartCount=0`, from 22:45:07 UTC.
- Monitoring: 55 systems, 54 healthy, 0 unknown. `lucos_backups` and `lucos_docker_health` both green.
- Interpreter confirmed as `3.14.6` in the running container, with the full dependency stack importing cleanly — `fabric` 3.2.3, `paramiko` 5.0.0, `invoke` 2.2.1, `requests` 2.34.2, `yaml` 6.0.3, `jwt` 2.13.0, `cryptography` 50.0.0, `jinja2` 3.1.6, and `charset_normalizer` **3.5.1**, the exact package that could not load before.
- Hourly cron paths exercised ad-hoc and confirmed against schedule-tracker, not merely against `/_info`: `config` age 21s / 0 errors, `tracking` age 7s / 0 errors.

- **`create-backups` executed end-to-end on the new interpreter**, which is the verification that actually mattered: this incident changed the Python minor version under the whole application, and a green `/_info` proves only that the web server imports — not that Fabric, Paramiko and rsync still work. The run created real archives and distributed them to every destination, exercising both backup strategies:

```
Creating local archive of <Volume lucos_photos_postgres_data on avalon>
Copying …lucos_photos_postgres_data.2026-08-17.tar.gz … to /share/backups/host/avalon/volume/ on aurora
Copying …                                              … to /srv/backups/host/avalon/volume/ on salvare
Copying …                                              … to /srv/backups/host/avalon/volume/ on xwing
Rsyncing snapshot of lucos_photos_photos … on aurora
```

**Caveat, so this isn't read as more than it is:** that ad-hoc run did not complete — it was killed about two-thirds through by the 540-second timeout on the command running it, not by any fault. So the mechanism is verified against every host and both strategies; a full 36-volume pass is not. It left nothing behind (the lock is `flock` on an open descriptor and died with the process; schedule-tracker still read `create-backups ok=true, errors=0`; no partial-state file survived), and the 03:25 scheduled run proceeded normally.

Worth recording, since it was found while checking that: `last_success` and `create.lock` live in `/var/run/lucos_backups/` **inside the container, with no volume mounted**, so every deploy wipes them and the next run always does a full backup rather than skipping. That behaviour is fail-safe — it errs toward backing up — so it is noted as a mechanism, not filed as a defect.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| **Restore:** revert base image to `python:3.14.6-alpine`, with a comment at the point of the mistake | lucas42/lucos_backups#391 | Done — merged 22:42 UTC |
| **Reproducibility:** `pipenv install` → `pipenv install --deploy` so the build installs what the lock pins and fails loudly when it can't | lucas42/lucos_backups#392 | Open |
| **CI guard:** run the test suite inside the built image — CI passed on this commit because `test` runs on `cimg/python:3.14`, not the artefact we ship. Validated against the broken image | lucas42/lucos_backups#393 | Open |
| **Estate-wide reproducibility:** convention requiring Python Dockerfiles to install from the lockfile — **8 of 8** Python repos currently re-resolve at build time | lucas42/lucos_repos#488 | Open |
| **Deploy containment:** a failed deploy leaves the service down, and re-running the last-green pipeline does not roll back | lucas42/lucos_deploy_orb#192 | Open |
| **Estate-wide:** decide the convention for base-image bumps that break at runtime. Fourth break of this class; the CI guard built for the third (lucas42/lucos_media_metadata_manager#386) was never rolled out | lucas42/lucos#273 | Awaiting Decision — evidence from this incident added as a comment |
| **Alert volume:** one CircleCI API outage sent 596 alert emails in a day against a 1–4 baseline; propose coalescing alerts that share a cause | lucas42/lucos_monitoring#302 | Open — same-day, adjacent rather than causal |
| **Crash-loop detection:** `lucos_docker_health` reported 42 recoveries during an outage that never recovered (Stage 5) | Proposed to team-lead for boarding — see below | Not yet filed |

Two things deliberately **not** filed, with reasons, so the decision is visible rather than silent:

- **A monitoring check for "a container is restarting".** `lucos_docker_health` already detected this correctly and went red at 07:42. Adding detection would not have shortened the outage by a minute — the gap was response, not detection. Building a second detector for a failure the first detector caught is how monitoring accretes maintenance tax without buying anything.
- **A guard on the two remaining pre-release repos** (`lucos_contacts_googlesync_import`, `lucos_media_weightings`, both `python:3.15.0rc1-alpine`). Tested against the exact wheel that broke this service; both import it cleanly, and CPython freezes its ABI at rc. Nothing is currently broken or at risk, so a repo-local guard now would pre-empt the estate decision in lucas42/lucos#273 with a fourth bespoke defence — which is precisely the situation that ticket exists to end.

On the Stage 5 crash-loop detection gap: the change I'd propose is that `lucos_docker_health` treat a **rising `RestartCount`** as unhealthy, independent of health status. `RestartCount` is monotonic, so comparing it between polls gives the same answer whenever you sample — it turns an oscillating instantaneous state into a level signal — and it needs one map in a loop that already polls every 60 seconds. (Requiring the counter to rise across two consecutive polls avoids flagging an ordinary one-off restart.) There may also be no code needed at all for the amplifier half: the "any of the last N runs" window is already per-job — `docker_health/avalon` uses 5, `backups/tracking` uses 3, `backups/create-backups` uses 2 — so setting docker_health's to 1 may remove it by configuration. Not filed as a ticket by me; passed to team-lead to board and route, since it is a defect in one service rather than an estate convention question.

The response-time gap in Stage 6 is the one finding I'd expect to generate a follow-up and have not filed one for. The available remedies — more frequent ops checks, escalation on repeated unacknowledged alerts, or accepting that a personal estate has no on-call — are a judgement call about how much this estate should cost to run, not a defect with a correct fix. Raising it here for lucas42 to direct rather than filing a ticket that presumes the answer.

---

## Lessons

- **A latent base-image bump is worse than a breaking one.** The three previous incidents in this series broke on the deploy that caused them, which at least put the cause next to the effect. This one was armed for 11 days and fired on a third party's release schedule, with the triggering commit — a GitHub Actions bump — bearing no relationship to the failure.
- **If the build re-resolves, the lockfile is a comment.** Any commit can change the dependency set, "revert the dependency bump" stops being a valid remediation, and the committed state stops describing the deployed state.
- **A failed deploy is not a safe deploy.** `docker compose up` destroys the working container before it discovers the new one is broken. Red CI here meant production was already down.
- **Re-running an old pipeline is not a rollback** on this estate — the version is resolved from the newest git tag, not the pipeline's commit.
- **Detection and response are different systems, and we only have one of them.** Every alerting component worked correctly and the outage still ran 15h24m. The 2026-07-31 report's "quiet failures are the expensive ones" needs qualifying: this failure was loud in every available channel and lasted longer than the quiet one.
- **To watch something that oscillates, compare a monotonic counter — don't sample the state.** Sampling gives a phase-dependent answer, and a tolerance rule ("any of the last N succeeded") then converts one lucky sample into a green check. A false recovery is worse than silence: it argues the problem is fixing itself.
- **A guard keyed to the wrong clock is absent exactly when it's needed.** `lucos_docker_health`'s stuck-starting threshold exists to catch a container that never comes up, and a container restarting every 4.5 seconds resets its clock faster than it can fire.
- **Reason from the artefact, not the commit log.** The commit history pointed convincingly at the `charset-normalizer` bump. Reading the lockfile and the installed package *out of the built image* took one command and pointed somewhere else entirely.

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.

No credentials were exposed, rotated, or involved. The failure was a dependency ABI mismatch. The data-protection risk was one of *absence* — no backups ran for 15h24m and the 15:25 safety-net run was lost — not of exposure. No existing backup data was lost, deleted or corrupted. `lucos-security` reviewed this draft and agreed with the "No" classification: an availability incident, not a confidentiality or integrity one.

That said, the Stage 3 finding has a security reading that is *not* redacted-sensitive but should not be lost in a reproducibility framing: the absent code-review gate between PyPI and production described above is a standing supply-chain exposure on the service that holds production host credentials. It is public, tracked in lucas42/lucos_backups#392 and lucas42/lucos_repos#488, and needs no advisory handling — exploiting it requires an upstream package compromise first.
