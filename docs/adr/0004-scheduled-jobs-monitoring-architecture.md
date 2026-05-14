# ADR-0004: Scheduled-jobs monitoring architecture

**Date:** 2026-05-13
**Status:** Accepted
**Discussion:** https://github.com/lucas42/lucos/issues/140

## Context

`lucos_schedule_tracker` is the central liveness tracker for scheduled jobs across the estate. The current shape is:

- Each cron job calls `POST /report-status` on schedule_tracker with `{system, frequency, status, message?}` at the end of every run.
- schedule_tracker persists `(system, frequency, last_success, last_error, error_count, message)` in SQLite (`schedule_v2`).
- schedule_tracker derives, on the server, an alert threshold from `frequency` (the rule has a step-change at 4 days — see `src/database.rb#calculate_time_threshold`).
- schedule_tracker exposes the resulting check set on its own `/_info` payload. lucos_monitoring fetches that payload as part of its standard `fetcher_info` poll and renders every scheduled-job check under the `lucos_schedule_tracker` system row.

Three architectural questions surfaced in [lucas42/lucos#140](https://github.com/lucas42/lucos/issues/140) (verbatim from @lucas42's original ask):

1. Is `lucos_schedule_tracker` the right approach at all?
2. What works well in the current design, and what could be improved?
3. Should scheduled-job checks appear under each *owning* system's monitoring row, rather than all bundled together under `lucos_schedule_tracker`?

Two further architectural problems came out of the first-pass scan and are load-bearing for any answer to the above:

- **The `system` field on `/report-status` is overloaded.** For systems with multiple jobs, callers synthesise IDs like `lucos_arachne_ingestor_dbpedia_meanOfTransportation` by concatenating real-system + sub-job code with `_` as delimiter. Since `_` is a valid character in real system codes, **there is no reliable way to recover the owning system from the synthetic key.** This is why the [lucos_schedule_tracker#73](https://github.com/lucas42/lucos_schedule_tracker/issues/73) deploy-window-flap fix needs a hand-maintained `dependsOn` mapping rather than a derivation rule.
- **Bundling all scheduled-job checks under `lucos_schedule_tracker` in monitoring** means a flaky arachne-ingestor heartbeat shows up against `lucos_schedule_tracker` rather than `lucos_arachne`. The deploy-window suppression that monitoring already does (when `lucos_arachne` is being deployed, its own failing checks are quietly suppressed for the deploy window) does **not** apply to a check that is attributed to `lucos_schedule_tracker` — even when the check is morally a property of the arachne system. This is what produces the flaps `lucos_schedule_tracker#73` was trying to suppress with `dependsOn`.

Five further sub-questions were noted in the issue body. Each is answered below.

### What works well today

- **Central state.** Every cron job in the estate gets liveness tracking for free, without any of them needing a docker volume, backup integration, or configy volume entry. For ~20 cron callers across the estate, that is a meaningful cost saving in operational burden.
- **Push model fits cron.** Cron jobs are by nature short-lived; they have no long-running container to host a `/_info` endpoint. A push at end-of-run is the natural shape.
- **Uniform threshold rule.** A class problem (weekly jobs alerting too slowly, 5-minute jobs alerting too noisily) is fixed in one place by editing schedule_tracker's threshold function — no estate-wide rollout. This is the property called out in @lucas42's design pointer 2 and is correct.
- **Per-job retirement.** Removing a job is a `DELETE /schedule/{system}` call to schedule_tracker — no estate-wide config change.

### What does not work

- **Overloaded `system` field.** As above.
- **Wrong owning-system attribution in monitoring.** As above. This is the proximate cause of `schedule_tracker#73`.
- **Bundled `/_info` payload.** schedule_tracker's `/_info` is being asked to do two unrelated things: declare the health of schedule_tracker *itself* (the service), and emit a synthetic check per scheduled job *about other systems*. These should not share a payload.

## Decision

Keep schedule_tracker as the central state store for scheduled-job heartbeats. Split its responsibilities so the right pieces of information flow to the right places:

1. **schedule_tracker continues to receive heartbeats and persist last-run state.** This is the part that @lucas42's pointer 1 protects: per-system state is operationally expensive, and the central tracker is justified by that cost saving.
2. **schedule_tracker continues to derive alert thresholds server-side.** Per @lucas42's pointer 2; no system-specific reason to override has been raised.
3. **The `system` field on `/report-status` is de-overloaded** by introducing `POST /v2/report-status` accepting `{system, job_name, frequency, status, message?}`. `system` is the owning system (e.g. `lucos_arachne`); `job_name` is the sub-job code (e.g. `ingestor_dbpedia_meanOfTransportation`). **`job_name` is required and must be a non-empty string** — every caller picks a name for its job at migration time, even single-job callers. This forces the "what if I grow to multi-job tomorrow?" question to be answered up front rather than deferred, and keeps the API contract honest about the field carrying meaning. The server rejects missing / empty / null `job_name` with HTTP 400. Retirement of a single named job is via `DELETE /v2/schedule/{system}/{job_name}` (mirror of the existing v1 `DELETE /schedule/{system}`). v1's DELETE form continues to address the row keyed `(system, '')` and is the form used during the per-cron migration cleanup to remove orphaned synthetic-ID rows.
4. **schedule_tracker exposes a new endpoint `GET /jobs`** (machine-readable list of all known scheduled jobs, each tagged with its owning `system` and `job_name`, with the derived check and metrics already computed server-side).
5. **lucos_monitoring adds a new fetcher (`fetcher_scheduled_jobs`)** that polls schedule_tracker's `GET /jobs` and attributes each check to its *owning system*. Modelled on the [`fetcher_circleci`](https://github.com/lucas42/lucos_monitoring/blob/main/src/fetcher_circleci.erl) precedent from [lucos_monitoring#93](https://github.com/lucas42/lucos_monitoring/issues/93).
6. **schedule_tracker's own `/_info`** stops emitting derived scheduled-job checks. After the cutover it carries only checks intrinsic to the schedule_tracker service itself (DB liveness, etc).

### Why this shape, and not the alternatives

The issue body laid out three coherent designs (centralised / decentralised / hybrid) and three placements for the bundle/grouping fix (re-attribute via payload / per-system `/_info` / monitoring-side synthetic group). This ADR is the centralised design with the monitoring-side grouping fix expressed via a new fetcher.

- **Decentralised design (every system hosts its own scheduled-job state).** Rejected. Per @lucas42 pointer 1: this pushes a docker volume + backup integration + configy volume entry onto every cron-running system. For ~20 systems, that is a significant operational cost increase to solve a problem (correct attribution in monitoring) that has a cheaper fix on the monitoring side.
- **Monitoring re-attributes checks fetched from schedule_tracker's `/_info` by reading a per-check `system` field.** Rejected as a primary mechanism. This would conflate schedule_tracker's own `/_info` (the health of the *service*) with the synthetic checks it currently emits about other systems. The two have different consumers, different lifecycles, and different access semantics; they should not share a payload. A separate `GET /jobs` endpoint, consumed by a domain-specific fetcher, mirrors the `fetcher_circleci` precedent and keeps the two concerns separate.
- **A monitoring-side "synthetic group by `system`" feature that re-buckets checks declared at one source against a different display owner.** Rejected. The mechanism is generic-looking but would only have one consumer (this one), and would couple monitoring's display logic to schedule_tracker's payload shape forever. The `fetcher_scheduled_jobs` approach achieves the same outcome (correct attribution) without that coupling — each fetcher type already knows which system it is attributing to.

### Push vs pull

The decision is hybrid: **push on the cron→schedule_tracker side, pull on the schedule_tracker→monitoring side.**

This preserves what works:

- Cron jobs continue to push at end-of-run. A pull-based alternative (each cron exposing a status endpoint that monitoring polls) is the decentralised design rejected above.
- monitoring continues to poll. A push-based alternative (schedule_tracker pushing scheduled-job state to monitoring) would invert the existing direction of every other fetcher in monitoring; no win.

The push-side single-point-of-failure (schedule_tracker downtime ⇒ cron's heartbeat is dropped ⇒ false missed-heartbeat alert when the cron's threshold expires) is **not solved** by this redesign. See "Consequences → Negative" below.

### Where state lives

Single source of truth: schedule_tracker's SQLite (`schedule_v2`, evolved to include `job_name`). Both `POST /v2/report-status` writes and `GET /jobs` reads address the same rows. No state in monitoring (monitoring caches the latest fetch, as it does for every other fetcher; this is not authoritative state).

### Threshold derivation

Stays server-side in schedule_tracker, per @lucas42's pointer 2. The class-problem benefit (fix a weekly-job alert latency in one place, no estate-wide rollout) is the deciding criterion; no system-specific reason to override has been raised in the discussion.

Two narrower concerns about the existing rule are noted but **out of scope for this ADR**:

- The 4-day step-change in `calculate_time_threshold` (a job at 3d23h gets a ~12d threshold; bump it to 4d and it gets ~8d30m) is a UX hazard for any job near that boundary. A continuous formula could smooth this without moving control client-side. This is a separate, smaller change against schedule_tracker's threshold function, raised as a follow-up.
- A future per-job override (a job opts out of the standard rule with an `expected_within_seconds` field on `/v2/report-status`) is **not** part of this ADR. If the class-rule turns out to be wrong for one specific job, the right response is to refine the class-rule, not to escape it. If a coherent system-specific reason appears later, it can be added then.

### Grouping/bundling — the `/_info` boundary question

The redesign moves the grouping question to its right home: it is a monitoring concern, not a schedule_tracker concern. schedule_tracker emits a flat list of `(system, job_name, check, metrics)` rows; monitoring's `fetcher_scheduled_jobs` attributes each row to its `system` value when it calls `gen_server:cast(StatePid, {updateSystem, Host, system, Type, scheduled_jobs, Checks, Metrics})`. The check key within that system row is `job_name` (or a synthetic `scheduled-job` if `job_name` is absent — for single-job systems migrating from v1). Multiple jobs for the same system produce multiple check keys under that system's row, as expected.

### Knock-on for `dependsOn` and `schedule_tracker#73`

Once a scheduled-job check is attributed to its *owning* system, monitoring's existing deploy-window suppression machinery applies to it automatically: when `lucos_arachne` enters a suppression window (because it is being deployed), every check attributed to `lucos_arachne`, including its scheduled-job checks fetched from schedule_tracker, is suppressed for that window without any per-check `dependsOn` declaration.

This means [`lucos_schedule_tracker#73`](https://github.com/lucas42/lucos_schedule_tracker/issues/73) — which proposed adding ~30 hand-maintained `dependsOn` entries to schedule_tracker's `/_info` — is **superseded in principle by this ADR**: the hand-mapped workaround should not be implemented. However, the flap problem only fully disappears for any given cron caller once that caller has migrated from v1 (where its `system` field is a synthetic concatenation that does not match the owning-system suppression key) to v2 (where the `system` field is the real owning system). For callers still on v1 on flag-day, the flap is still latent — their checks still attribute to the synthetic-ID string, not to a real owning system. So:

- On flag-day (monitoring↔schedule_tracker cutover): mark `lucos_schedule_tracker#73` as **superseded in principle by ADR-0004**; do not implement.
- Close `lucos_schedule_tracker#73` as **solved** only when the last v1 caller has migrated to v2 (i.e. at the same point v1 can be removed).

The polymorphic `dependsOn:list` work from [`lucos_monitoring#227`](https://github.com/lucas42/lucos_monitoring/issues/227) is still valuable for genuinely cross-cutting checks (the loganne `webhook-error-rate` motivating case), but is **not needed for scheduled-jobs** under this design — each scheduled-job check has exactly one owning system, not a list of them.

### Other "N similar things under one row" cases

Three other estate systems share the same general shape — N similar checks bundled under a single owning service:

- `lucos_arachne_ingestor` — multiple ontology source loaders, currently bundled under one monitoring row.
- `lucos_backups` — multiple backup targets and modes.
- `lucos_media_import` — multiple source feeds.

The architectural language this ADR establishes — *checks attributed to their owning system, possibly via a domain-specific fetcher* — applies to all three. They are **explicitly out of scope** here: each has its own owner-system relationship and its own set of consumers, and folding them into this ADR would broaden the design surface unhelpfully. They are named so the next person reading this document knows the same idea applies elsewhere and does not need to rebuild it from scratch.

### lucos_schedule_tracker_pythonclient scope

The python client gets a new function for v2 callers (passing `system` and `job_name` separately) but does **not** absorb scope it has been protected from before (third-party API wrappers, per-job retry logic). This is consistent with [`lucos_schedule_tracker#70`](https://github.com/lucas42/lucos_schedule_tracker/issues/70).

### Migration

Two interfaces, two strategies, per @lucas42's design pointer 4.

**Monitoring ↔ schedule_tracker (flag-day cutover, 2 systems).** Order:

1. schedule_tracker ships `GET /jobs` alongside the existing `/_info` payload (parallel, both present).
2. lucos_monitoring ships `fetcher_scheduled_jobs` reading from `GET /jobs`.
3. On a flag day:
   - lucos_monitoring's configuration is updated so the new fetcher is active.
   - schedule_tracker's `/_info` stops emitting derived scheduled-job checks.
4. Inside the same flag day, `lucos_schedule_tracker#73` is marked as **superseded in principle by ADR-0004** (the hand-mapped `dependsOn` workaround is not implemented). It is closed as "solved" later — see the schedule_tracker↔all-systems migration below.

**schedule_tracker ↔ all systems (v1/v2 parallel-run, ~20 cron callers).** Order:

1. schedule_tracker ships `POST /v2/report-status` accepting `{system, job_name, frequency, status, message?}`. `/report-status` (v1) continues to accept its existing body shape unchanged.
2. The DB schema gains a nullable `job_name` column. The primary key becomes `(system, COALESCE(job_name, ''))` so v1 calls (which omit `job_name`) continue to address the same row as before.
3. lucos_schedule_tracker_pythonclient gains a new function that uses `/v2`. The existing function stays callable and stays pointed at `/v1`.
4. Cron callers migrate piecemeal whenever they're next touched. Each migration is a one-line change in the cron's call site, **plus** a clean-up step: every per-cron migration PR must also issue `DELETE /schedule/{old_synthetic_id}` against schedule_tracker to remove the orphaned v1 row. This step is non-optional. Without it, the v1 row's `last_success` will go stale, the time threshold will expire, and `fetcher_scheduled_jobs` will emit a missed-heartbeat alert attributed to the synthetic-ID string *as if it were an owning system* — producing a visible "system" in monitoring that does not exist anywhere else in the estate. **Per-cron migration tickets are not raised now** — per @lucas42's procedural note in #140, they create board clutter while blocked. They are raised one-at-a-time as cron systems are touched after v2 is dual-running in production.
5. Once telemetry shows zero v1 callers remaining, v1 is removed in a subsequent change (separate ADR or close-out comment on the v2 implementation ticket). At this point `lucos_schedule_tracker#73` is closed as "solved" — the flap it described is structurally impossible once no synthetic-ID rows remain.

The two interface migrations are sequenced **independently**. The monitoring↔schedule_tracker side can land before any cron caller migrates to v2 — schedule_tracker's `GET /jobs` reads existing v1-shaped rows fine (it just reports `job_name: null` for those entries; the monitoring fetcher uses `scheduled-job` as the check key for them).

## Consequences

### Positive

- **Correct attribution in monitoring.** Scheduled-job checks appear under their owning system row, which is where users go looking for them.
- **Deploy-window flaps disappear by construction *for v2-migrated callers*.** No hand-maintained `dependsOn` map; the existing suppression machinery handles it because the check is attributed to the right system. The flap latency for un-migrated v1 callers is unchanged — they keep flapping on their owning system's deploy windows until they migrate.
- **Overloaded `system` field is fixed.** Sub-job names are recoverable from the API contract, not from string-split heuristics over an ambiguous delimiter.
- **schedule_tracker's `/_info` becomes honest.** It carries only the health of the schedule_tracker service itself, not synthetic checks about other systems.
- **No new infrastructure cost.** No new container, no new volume, no new backup integration. The fetcher is added to an existing monitoring binary; the v2 API is added to an existing schedule_tracker binary.
- **Class-problem property preserved.** Threshold tuning is still a single-place edit; the central tracker still earns its keep.
- **The architectural language extends.** Other "N similar things under one row" cases (arachne ingestor, backups, media_import) can use the same pattern (domain-specific fetcher attributing to owning system) without rebuilding the framing.

### Negative

- **Push-side single point of failure is unchanged.** A cron run whose `POST /report-status` (or `/v2/report-status`) call happens while schedule_tracker is down has its heartbeat dropped; the missed-heartbeat alarm will fire spuriously later, when the threshold expires. Mitigations considered and rejected:
  - Per-system state hosting (the decentralised design) — rejected above.
  - Pull-from-cron (monitoring polls each cron's `/_info`) — same rejection; crons have no long-running container.
  - Cron-side retry against a queue — adds operational complexity (the queue) to solve an infrequent failure mode.
  The accepted mitigation is the existing `failThreshold` machinery on the derived check (`error_threshold` in `calculate_error_threshold`, which already gives high-frequency jobs more tolerance) plus monitoring schedule_tracker's own uptime, which is already done.
- **Two report-status APIs during the migration window.** Slight maintenance overhead until the last v1 caller migrates. Mitigated by treating v1 as a thin shim over v2 in the implementation (v1 writes the same row v2 would, with `job_name = null`).
- **A new monitoring fetcher is one more thing to think about.** Mitigated by following the `fetcher_circleci` template closely — same shape, same lifecycle, same attribution mechanism.
- **The 4-day step-change in the threshold function is not fixed by this ADR.** It is raised as a separate follow-up to avoid widening scope; jobs configured close to the boundary continue to be exposed to it until that follow-up lands.
- **Synthetic check key for v1-shaped rows.** A v1 caller (`{system}` only, no `job_name`) produces a single check with key `scheduled-job` under that system's row. If a system later starts using v2 with a `job_name`, it will get a second check key for the named job; the v1-derived `scheduled-job` key will linger until that system's last v1 caller migrates (or until the row is explicitly deleted). This is acceptable — it is visible behaviour in monitoring, not hidden state — but worth flagging so it isn't a surprise during the migration.
- **Per-cron migration carries a non-optional clean-up step.** Each migration PR must also `DELETE /schedule/{old_synthetic_id}` to remove the orphaned v1 row. Forgetting this step leaves a stale row whose threshold-expiry surfaces as a missed-heartbeat alert attributed to the synthetic-ID string as if it were an owning system — a phantom system on the monitoring dashboard. The accepted mitigation is to call this out in every per-cron migration ticket and to leave it noted on the v2 implementation ticket as a migration-checklist item. Forgetting once is recoverable (delete the row manually); the failure mode is loud rather than silent.

### Follow-up actions

Per the procedural note in [lucas42/lucos#140](https://github.com/lucas42/lucos/issues/140), implementation tickets are limited to those that have a target interface to migrate to. Three are raised from this ADR:

- **`lucos_schedule_tracker`** — `POST /v2/report-status` with `{system, job_name, frequency, status, message?}`; DB schema migration adding `job_name` and updating the primary key; v1 endpoint preserved as a shim writing `job_name = null`.
- **`lucos_schedule_tracker`** — `GET /jobs` endpoint returning a machine-readable list of all known scheduled jobs (each tagged with `system`, `job_name`, `check`, `metrics`) for monitoring to consume.
- **`lucos_monitoring`** — `fetcher_scheduled_jobs` polling `GET /jobs` and attributing each check to its owning system, modelled on `fetcher_circleci`.

Deferred until v2 is dual-running in production (per @lucas42's procedural note):

- Per-cron-caller migration tickets (~20 of them, raised one-at-a-time as cron systems are touched).
- Removal of `/report-status` (v1) and the `job_name`-nullable schema accommodation, in a subsequent change.

Out of scope of this ADR but flagged for follow-up:

- Smoothing the 4-day step-change in `calculate_time_threshold` to a continuous formula. Separate, smaller change against schedule_tracker.
- Same architectural pattern (owning-system attribution, possibly via domain-specific fetcher) for `lucos_arachne_ingestor`, `lucos_backups`, and `lucos_media_import`. Each is its own design step.
- [`lucos_schedule_tracker#73`](https://github.com/lucas42/lucos_schedule_tracker/issues/73) (the hand-mapped `dependsOn` proposal): mark as **superseded in principle by ADR-0004** when the monitoring-side cutover lands; close as **solved** when the last v1 caller has migrated. No code work needed on #73 itself.

## Amendments

### 2026-05-13: `job_name` required and non-empty on `/v2/report-status`

The original Decision #3 wording was ambiguous about whether `job_name` could be absent or empty on v2 calls (the DB schema allowed `(system, NULL)` rows to accommodate v1, and the wording did not rule out v2 doing the same). After the v2 producer side shipped and the migration tickets were filed, the design was tightened: `job_name` is **mandatory and non-empty** on `/v2/report-status`. Decision #3 has been updated in place to reflect this.

Reasoning (per @lucas42): a single-job system today may grow to multi-job tomorrow. Forcing every caller to name its one job at migration time makes that future transition trivial (just add a second name, no schema migration, no API contract change). It also keeps the field meaningful — an optional-defaults-to-empty field invites callers to skip thinking about naming, which is exactly the laziness we want to design out.

Implementation: API change tracked in `lucas42/lucos_schedule_tracker#85`. Pythonclient mandatory-`job_name` was already pivoted in `lucas42/lucos_schedule_tracker_pythonclient#41` review. All 6 single-job migration tickets that previously suggested `job_name=""` have been updated to suggest a concrete name.
