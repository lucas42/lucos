# Incident: EXIF orientation fix triggers cascade of data loss, deploy failures, and backup outages

| Field | Value |
|---|---|
| **Date** | 2026-03-16 to 2026-03-17 |
| **Duration** | ~17 hours (first impact ~09:00 UTC 2026-03-16 to monitoring cleared ~01:15 UTC 2026-03-17) |
| **Severity** | Data loss (face-person assignments); service degradation (lucos_photos deploy blocked ~2 hours; lucos_backups tracking broken ~12 hours) |
| **Services affected** | lucos_photos, lucos_backups, lucos_monitoring |
| **Detected by** | SRE investigation following user report of missing photo-person associations |

---

## Summary

A fix for EXIF image orientation (lucas42/lucos_photos#201) required a bulk reprocess of all 758 photos in the library. The reprocess triggered face detection on all photos, which deleted and recreated all face records — including manually confirmed face-person links. 261 of 628 persons lost all photo associations. Recovery required restoring the PostgreSQL volume from backup, but the restore process (using `docker run` rather than `docker compose`) created a volume without Docker Compose labels. This caused: (1) a subsequent deploy to fail with "container name already in use"; and (2) the lucos_backups tracking script to crash on every run, taking down volume tracking for all three production hosts. Both were resolved, but required two follow-up code fixes and a manual sysadmin operation to recreate the volume with correct labels. The incident also surfaced a gap in documentation of the telemetry API that delayed diagnosis of a related Android sync issue.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-03-16 ~09:00 | Bulk reprocess of 758 photos triggered to apply EXIF orientation fix from lucas42/lucos_photos#201. First run short-circuits — `process_photo` idempotency check finds thumbnails on disk and skips regeneration. |
| 2026-03-16 ~09:15 | All 758 thumbnails deleted from derivatives volume. Bulk reprocess re-triggered. Jobs complete successfully; EXIF-corrected thumbnails generated. |
| 2026-03-16 ~09:30 | User reports persons are missing photo associations. Investigation reveals `detect_and_save_faces` deleted all face records — including `person_confirmed=True` — on every photo. 261/628 persons now have zero face records. |
| 2026-03-16 ~10:00 | lucos_photos#208 raised (code fix: preserve confirmed links). lucos_photos#209 raised (data recovery: restore from 2026-03-15 backup). |
| 2026-03-16 (afternoon) | lucos-system-administrator restores `lucos_photos_postgres_data` volume from 2026-03-15 backup using `docker run` with alpine container. Volume recreated without Docker Compose labels. |
| 2026-03-16 ~23:21 | lucas42/lucos_photos#207 (Loganne fix) and #208 (preserve confirmed links) merged and deploy triggered. Deploy fails: "container name already in use" for `lucos_photos_postgres`. Restore left postgres container running; deploy tries to create a new container with the same name. |
| 2026-03-17 ~00:26 | Loganne `plannedMaintenance` event sent. `lucos_photos_postgres` container manually stopped and removed. Deploy workflow retried. All jobs succeed. |
| 2026-03-17 ~00:42 | lucos_backups monitoring alerts investigated. Root cause: `Volume.__init__` in lucos_backups crashes with "not enough values to unpack" when `data["Labels"]` is empty — caused by the unlabelled volume from the restore. PR lucas42/lucos_backups#62 raised and merged. |
| 2026-03-17 ~01:00 | PR #62 deployed. New error: `NameError: name 'project' is not defined` on all three production hosts — regression introduced by #62 removing a variable assignment that was still used downstream. PR lucas42/lucos_backups#63 raised and merged. |
| 2026-03-17 ~01:05 | PR #63 deployed. Tracking refresh triggered. All monitoring alerts clear. |
| 2026-03-17 ~01:10 | User reports 3 photos from 2026-03-16 not appearing in lucos_photos. Telemetry investigated. Gap in telemetry on 2026-03-16 initially interpreted as evidence the app didn't run — later realised the telemetry table was also wiped by the DB restore. |
| 2026-03-17 ~01:15 | `lucos_photos_postgres_data` volume recreated by sysadmin with correct Docker Compose labels (live data copied, not restored from backup). All monitoring clear. |

---

## Why Traditional Root Cause Analysis Doesn't Fit Here

This incident does not have a single root cause. It is a chain of individually reasonable decisions and pre-existing gaps that compounded into a multi-service cascade. The EXIF reprocess was intentional. The data loss was a known limitation of `detect_and_save_faces` that nobody had documented. The restore method was sensible. The volume label loss was an undocumented side effect of that method. Each step in the chain was a surprise only in the context of what came before.

The more useful question is: which contributing factors, if addressed, would have prevented the most downstream damage?

---

## Chain of Events and Contributing Factors

### Stage 1: Bulk reprocess wipes confirmed face-person links

**What happened:** `detect_and_save_faces` unconditionally deletes all existing face records before recreating them. This includes records where `person_confirmed=True`, which represent manually curated human-verified links. Running it on all 758 photos destroyed months of curation work.

**Contributing factors:**

- **`detect_and_save_faces` lacked a preservation path for confirmed links.** The function was written for initial processing, not for reprocessing already-curated data. There was no distinction between "this face was auto-assigned" and "a human confirmed this." Fix tracked in lucas42/lucos_photos#208 (now merged).

- **No warning before triggering a destructive bulk operation.** The reprocess was triggered without any upfront check of how many confirmed links existed or any prompt to confirm the operation was safe. A pre-flight check — "this will re-run face detection on N photos and may affect M confirmed links, continue?" — would have surfaced the risk.

- **The idempotency short-circuit masked the first attempt.** The initial bulk run completed in ~33ms because `process_photo` found thumbnails on disk and skipped processing. This required a workaround (deleting all thumbnails) that itself bypassed the system's normal cautious approach. The fact that a workaround was needed at all should have been a signal to pause and re-examine what else the reprocess might affect.

---

### Stage 2: Database restore creates volume without Docker Compose labels

**What happened:** Restoring the PostgreSQL volume using `docker run` with an alpine container creates a new Docker volume, but without the `com.docker.compose.project`, `com.docker.compose.version`, or `com.docker.compose.volume` labels that `docker compose` normally applies. This is an undocumented side effect of the restore method.

**Contributing factors:**

- **The restore procedure was not documented.** There was no runbook for restoring a lucos_photos database volume. The sysadmin used a reasonable general-purpose Docker technique, unaware that it would strip labels. A documented restore procedure using `docker compose up` to create the volume first (ensuring labels are applied) would have prevented this.

- **Docker Compose labels are required by lucos_backups but this dependency was invisible.** Nothing in the lucos_photos repo, the lucos_backups repo, or any shared documentation explained that volumes must have Compose labels or that lucos_backups would crash without them. This was implicit tribal knowledge.

- **lucos_backups crashed the entire host's tracking on a single bad volume.** Rather than skipping the unlabelled volume with a clear warning, `Volume.__init__` let the crash propagate, taking down tracking for all volumes on that host. Fixed in lucas42/lucos_backups#62 and #63.

---

### Stage 3: Restore leaves postgres container running, blocking deploy

**What happened:** The restore process left the `lucos_photos_postgres` container running. When the CI deploy later ran `docker compose up`, it tried to create a new container with the name `lucos_photos_postgres`, conflicting with the already-running one. Every deploy attempt failed with "container name already in use."

**Contributing factors:**

- **The restore process had no documented teardown step.** After restoring data into a volume, the procedure left an ephemeral container running. A documented restore runbook would include stopping and removing the temporary container as a final step.

- **The error message was misleading.** "Container name already in use" sounds like a stopped/orphaned container issue. The running container had 36 minutes of uptime and was visibly `Up` in `docker ps`, but initial diagnosis focused on a stopped container that had "mysteriously disappeared." The correct diagnosis — that the currently running container was the conflict — required a second look after user pushback.

- **`docker compose up` gives no indication of which running container is blocking it.** A more descriptive error noting the conflicting container's ID and uptime would have pointed to the correct diagnosis immediately.

---

### Stage 4: lucos_backups code fix introduces regression

**What happened:** PR lucas42/lucos_backups#62 fixed the label parsing crash but inadvertently removed the `project = labels['com.docker.compose.project']` variable assignment while consolidating the error handling. The variable was still used 15 lines later to populate `self.data`, causing a `NameError` on every volume with labels — breaking tracking on all three hosts.

**Contributing factors:**

- **Only the changed lines were reviewed, not the downstream usage.** The fix was reviewed and approved as correct in isolation. Neither the author nor the reviewer scrolled down to check what used the removed variable. Reading the full function before editing any part of it would have caught this.

- **No automated tests for `Volume.__init__`.** A unit test for the happy path (volume with labels) would have caught the `NameError` before the PR was merged. The test suite covered other paths but not this one.

- **The fix was written and merged under pressure** (monitoring alerts were red, user was watching). Time pressure is a classic factor in introducing regressions while fixing incidents.

---

### Stage 5: Telemetry gap misread as evidence the app didn't run

**What happened:** After the restore, investigation of 3 missing photos found no telemetry events for 2026-03-16. This was initially interpreted as the Android app not running that day. In fact, the telemetry table is in the same PostgreSQL database that was restored from the 2026-03-15 backup — any telemetry from Sunday was wiped along with the photo records.

**Contributing factors:**

- **No documentation of where telemetry data lives or how to access it.** The SRE agent had to rediscover the telemetry API endpoint, authentication mechanism, and the fact that it uses the Android app's production key rather than the agent key. This wasted time and led to an initial incorrect reading of the data. Now documented in SRE agent memory.

- **The telemetry table shares the same database as application data.** This means a database restore for data recovery also silently wipes the telemetry history for the same period. The gap in telemetry is indistinguishable from "app didn't run" unless you know about the restore. Separating telemetry into its own database or table with a different retention/backup strategy would make this less ambiguous.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Preserve `person_confirmed=True` links in `detect_and_save_faces` | lucas42/lucos_photos#208 | Merged |
| Document volume restore procedure and Docker Compose label requirement | lucas42/lucos_backups#64 | Open |
| Document telemetry API access in a reference file | Captured in SRE agent memory | Done (reference doc to follow) |
| Investigate 3 missing photos from 2026-03-16 | lucas42/lucos_photos_android#74 | Open |
| Consider separating telemetry storage from application database | lucas42/lucos_photos#211 | Open |
| Add unit tests for `Volume.__init__` with and without labels | lucas42/lucos_backups#65 | Open |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.
