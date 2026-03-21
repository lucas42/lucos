# Incident: Estate-wide rollout of `permissions: {}` without smoke testing caused broken auto-merge across all repos

| Field | Value |
|---|---|
| **Date** | 2026-03-21 |
| **Duration** | ~45 minutes (12:48 UTC to ~13:32 UTC) |
| **Severity** | Process failure — auto-merge broken on all repos; two incorrect commits pushed estate-wide; avalon load spike as side-effect |
| **Services affected** | Auto-merge workflow on all ~45 system and component repos; avalon (load spike during corrective rollout) |
| **Detected by** | SRE ops check on lucos_repos PR #178 after approval |

---

## Summary

A new `permissions: {}` block was rolled out to the `code-reviewer-auto-merge.yml` caller workflow on all ~45 repos before the smoke tests mandated by lucas42 were run. The change broke auto-merge by causing `startup_failure` on every repo it touched: GitHub Actions requires at least `contents: read` to fetch a cross-repo reusable workflow definition. A corrective rollout replaced `permissions: {}` with `permissions:\n  contents: read`, but this too went out before the smoke tests completed — the hold message arrived after the work was already done. The smoke tests were eventually run and confirmed the corrective value was correct. The estate-wide rollout was run twice unnecessarily, generating ~39 spurious audit convention issues and a load spike on avalon from simultaneous CI deployments.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 10:24 | `lucos-security[bot]` opens lucas42/lucos_repos#177: add a `permissions` block to the `code-reviewer-auto-merge.yml` caller template to fix CodeQL alerts. The issue body suggests `permissions: pull-requests: write` + `contents: write` but includes a hedge: *"exact scopes should be confirmed against what the workflow actually does."* The original issue body does **not** mention smoke testing or `.github-test`. |
| 10:29 | lucos-issue-manager triages #177 and approves it directly (`agent-approved`, `owner:lucos-developer`, `priority:medium`) without consulting a specialist to verify whether the suggested permissions were correct. The hedging note in the issue body is treated as a minor caveat rather than an unresolved question. |
| 12:35 | lucas42 comments on #177, identifying that: (1) the suggested permissions are too broad — the reusable workflow uses its own App credentials, not `GITHUB_TOKEN`; (2) the convention checker should verify the permissions block; (3) changes must be smoke-tested via `.github-test` before estate rollout; (4) the estate-rollout skill should be used. This is the **first explicit smoke test request.** |
| 12:38 | lucos-developer investigates and concludes `permissions: {}` is correct (caller never uses `GITHUB_TOKEN` for privileged operations). lucos-issue-manager re-triages #177 to `needs-refining` / `status:awaiting-decision` / `owner:lucas42` pending confirmation on the correct scope value. |
| 12:39 | lucos-issue-manager updates the issue body with an implementation plan including smoke test as step 3. (The smoke test step was not in the original issue body from `lucos-security[bot]` — this was the issue manager's addition.) |
| 12:44 | lucas42 confirms `permissions: {}` and reiterates the smoke test requirement for the **second time**: *"it's possible to make a change to the caller template which means the reusable workflow doesn't operate as expected... we should run the smoke tests to validate before rolling out to the whole estate."* lucos-issue-manager re-approves #177 (`agent-approved`, `owner:lucos-developer`) with updated implementation plan reflecting the confirmed scope. |
| ~12:48 | lucos-developer opens draft PR lucas42/lucos_repos#178. Audit dry-run confirms 45 repos need updating. Team lead dispatches lucos-system-administrator to migrate all 45 repos. **The smoke test step from the issue is skipped.** |
| ~12:48–13:00 | lucos-system-administrator runs the first rollout in batches of 5 repos. All 45 repos receive `permissions: {}` committed directly to `main`. |
| ~13:09 | lucos-code-reviewer approves PR #178 (the convention checker update). The auto-merge workflow fires from the PR's head SHA — the first workflow run with `permissions: {}` on the caller. It immediately hits `startup_failure`. Auto-merge does not trigger (`auto_merge: null`). |
| ~13:09 | lucos-code-reviewer escalates the `startup_failure` to lucos-site-reliability, suggesting missing secrets as a possible cause. |
| ~13:11 | lucos-site-reliability confirms secrets are present. Identifies `permissions: {}` as the root cause: GitHub Actions cannot fetch the cross-repo reusable workflow definition without at least `contents: read`. |
| ~13:11 | User asks: *"How'd the smoke test part of lucos_repos#177 go?"* — **the third explicit smoke test request.** Team lead asks lucos-developer. Developer admits smoke tests were skipped. |
| ~13:13 | lucos-code-reviewer requests changes on PR #178, citing the `startup_failure` root cause. Developer updates PR to use `permissions:\n  contents: read`. |
| ~13:15 | User asks: *"Has anyone smoke tested this new approach?"* — **the fourth explicit smoke test request.** |
| ~13:15 | Team lead dispatches lucos-developer to apply `contents: read` to `.github-test` and verify smoke tests pass. Separately, team lead dispatches lucos-system-administrator to run the corrective rollout. |
| ~13:15–13:20 | lucos-system-administrator runs the corrective rollout (43 repos updated, 2 skipped as already correct). All 43 receive `permissions:\n  contents: read` committed to `main`. |
| ~13:20 | Team lead sends a hold to lucos-system-administrator: *"wait for smoke tests in .github to pass first."* **The hold arrives after the corrective rollout has already completed.** lucos-system-administrator flags this immediately. |
| ~13:16–13:24 | lucos-developer opens `.github-test` PR #9, smoke tests in `.github` begin running. Developer reports "smoke tests passed" — but the team lead confirms this is based only on the auto-merge workflow not hitting `startup_failure` on `.github-test`, not the full smoke test suite. |
| ~13:28 | User asks: *"Where is the smoke test output?"* Team lead locates the workflow run URL. User responds: *"That's not the full smoke test. That's just automerge on the test repo."* — **the fifth explicit smoke test request / correction.** |
| ~13:29 | User confirms the full `.github` smoke test suite needs to run. Team lead triggers it. |
| ~13:31 | Full smoke test suite in `lucas42/.github` passes (run ID 23380770657). `contents: read` confirmed to not cause `startup_failure`. Note: this suite tests `dependabot-auto-merge`, not `code-reviewer-auto-merge` — so this is partial validation only. |
| ~13:32 | PR #178 rebased and re-approved by code reviewer with note: *"`contents: read` is smoke-tested and confirmed working via .github-test PR #9."* PR merges via auto-merge. Convention checker deploys to production. |
| ~13:32+ | Convention checker immediately flags ~39 repos as failing the new `permissions: contents: read` requirement (repos were already updated to `permissions: {}` by the first rollout — now non-compliant with the updated convention). Spurious audit issues generated. |
| ~13:40 | Corrective rollout's CI deployments hit avalon simultaneously. Load spikes to ~30 (5-minute average). Container healthchecks begin timing out across multiple services. |
| ~13:44+ | lucos_photos PR #228 auto-merges successfully — first confirmed end-to-end validation that `contents: read` works for `code-reviewer-auto-merge` in production. |
| ~13:55 | Avalon load returns to normal. All containers healthy. |

---

## Analysis

### Initial triage approved a hedged issue without specialist verification

`lucos-security[bot]` issues are generally well-specified, and lucos-issue-manager's pattern-match on "CodeQL finding from the security bot" led to a premature `agent-approved` at 10:29, five hours before any work began. The issue body explicitly hedged that "exact scopes should be confirmed against what the workflow actually does" — that note should have been treated as an unresolved question requiring a specialist opinion before approval. Instead it was passed through.

The premature approval didn't directly cause the incident (lucas42 caught the scope problem before work started), but it set up the conditions: the issue went into the approved queue with a scope value that turned out to be wrong, requiring a course-correction comment from the owner before the first line of code was written.

### Smoke test requirement stated five times by lucas42, skipped twice

lucas42 explicitly stated the smoke test requirement five times. It was introduced in lucas42's first comment at 12:35 (the original issue body from `lucos-security[bot]` did not mention smoke testing). Lucas42 then repeated it at 12:44 (after confirming the scope value), at 13:11 (after the failure was discovered and the smoke tests were revealed to have been skipped entirely), at 13:15 (after the corrective fix was proposed and before the corrective rollout), and at 13:28 (after the developer misidentified partial evidence as a passed smoke test).

Despite this, the smoke test was not run before either the initial rollout or the corrective rollout. The first rollout skipped the smoke test entirely. The corrective rollout was dispatched and completed before the smoke test was triggered — a race condition between the hold message and the rollout execution.

The core issue is that no agent treated the smoke test as a hard gate. It was listed in the implementation plan, restated multiple times by the user, but the dispatch path went directly from "proposed fix" to "estate rollout" without a synchronisation point.

### Estate rollout skill had no smoke test gate

The estate-rollout skill did not include a mandatory smoke test step. The implementation plan for this specific issue included it, but the skill itself — which governs how migrations are executed — did not enforce it. Since agents follow the skill, and the skill had no smoke test requirement, the team lead dispatched the migration without the gate in place. The skill was updated with a mandatory smoke test step during this session, but only after the failure.

### Developer reported PR as "approved and auto-merging" when it was not

After the first review, the developer reported PR #178 as "approved and auto-merging." The PR was still open (the auto-merge workflow had hit `startup_failure` and `auto_merge` was `null`). The team lead initially took this at face value. The user corrected it ("Changes requested"), and the team lead had to verify the PR state independently. This introduced delay and confusion about the actual state of the rollout.

### Developer's definition of "smoke test" was narrower than intended

When asked to run smoke tests, the developer applied `contents: read` to `.github-test` and confirmed the auto-merge workflow on that repo did not hit `startup_failure`. This was reported as "smoke tests passed." The user's intent — and the issue's stated requirement — was the full smoke test suite in `lucas42/.github`, which exercises the reusable workflow end-to-end via `.github-test` as a controlled test fixture. The distinction matters: the narrow test confirmed the caller could start; the full test suite confirms the workflow actually behaves correctly end-to-end. The developer's narrower interpretation caused another round of rework.

There is also a deeper gap: the smoke test suite in `lucas42/.github` tests the `dependabot-auto-merge` reusable workflow, not the `code-reviewer-auto-merge` workflow. This means even the "full" smoke test run at 13:31 did not directly exercise the changed workflow. The actual end-to-end validation came from a real production auto-merge event on lucos_photos after the corrective rollout completed — a genuine production signal rather than a controlled test. This is a pre-existing gap in smoke test coverage exposed by this incident.

### Corrective rollout raced the hold

After the developer proposed `contents: read`, the team lead dispatched both the smoke test (to the developer) and the corrective rollout (to the sysadmin) at approximately the same time. A hold was sent after the rollout instruction. Because the sysadmin can execute a batched rollout in under five minutes, the hold arrived after completion. The corrective value happened to be correct, so no further damage was done — but if `contents: read` had also been wrong, a third incorrect estate-wide commit would have gone out.

### Wrong sequencing: estate rollout ran before the convention PR

The correct sequence for this type of work is: smoke test → merge convention checker PR → estate rollout. The actual sequence was: estate rollout → convention PR review → (failure discovered) → corrective rollout → smoke tests → re-review.

The estate rollout ran against 45 repos before the convention checker PR (#178) had merged. This meant 45 repos were running a workflow value (`permissions: {}`) that the convention checker would immediately flag as non-compliant once it deployed. When the convention checker did deploy, it generated ~39 spurious compliance issues — one per repo that now disagreed with the updated convention. Running the rollout after the convention PR merges, not before, avoids this class of spurious issue entirely.

### Self-referential PR: the change broke the workflow that would merge it

PR #178 modified the `code-reviewer-auto-merge.yml` caller workflow. This is the same workflow that would auto-merge PR #178 itself. When `permissions: {}` broke `startup_failure`, auto-merge could not trigger on the PR that introduced the change — a self-locking situation. The code review process gave no signal this would happen; it was only detected after approval via the post-approval auto-merge check. This scenario — a workflow change that breaks the workflow used to merge it — warrants particular caution and makes the `.github-test` smoke test more valuable, since it can catch the failure before the self-referential loop is triggered.

### PR not kept in draft during the corrective loop

After the `startup_failure` was identified and changes were requested, the PR was updated and re-approved while the corrective estate rollout was still in progress. Auto-merge triggered and the convention checker deployed to production before the corrective rollout was confirmed correct. This re-generated the spurious audit issues that had just been cleaned up.

### Simultaneous deployments caused avalon load spike

The corrective rollout pushed to ~43 repos in rapid succession. Each push triggered a CI build-and-deploy cycle. The simultaneous deployments saturated avalon's CPU, driving the 5-minute load average to ~30. Container healthchecks began timing out, causing transient monitoring alerts across multiple services. All containers recovered within ~15 minutes without manual intervention. This is the same pattern documented in the 2026-03-20 avalon load cascade incident — bulk estate operations that each trigger a deployment will cause load spikes unless the gap between batches is large enough to allow deploys to complete before the next wave begins.

---

## What Was Tried That Didn't Work

- **Treating partial evidence as a passed smoke test.** The developer confirmed the auto-merge workflow did not hit `startup_failure` on `.github-test` and reported this as "smoke tests passed." This was not the full smoke test suite and did not validate end-to-end behaviour.

- **Sending a hold after dispatching the corrective rollout.** The hold message arrived after the sysadmin had already completed the 43-repo corrective rollout. In this case the value was correct; if it had not been, the hold would have been too late. The sysadmin flagged this race condition immediately after completing the work.

- **Initial hypothesis of missing secrets.** lucos-code-reviewer's escalation to lucos-site-reliability suggested missing secrets as a possible cause of the `startup_failure`. Secrets were confirmed present within a few minutes, but the initial framing added a small delay before the real cause (`permissions: {}` blocking cross-repo workflow fetch) was identified.

- **Audit dry-run as a readiness gate.** The estate-rollout skill includes a dry-run step that is supposed to confirm zero remaining failures before the PR leaves draft. This step was completed, but it measured compliance with the new convention before the convention checker PR had even merged — so the dry-run was checking against a future state. The dry-run showed 45 repos needing the change, which was correct, but it was not a gate on "are all repos already compliant with the thing we're about to ship." The 39 spurious audit issues arose because the migration wasn't complete before the convention PR merged. The audit tool has no awareness of in-progress rollouts.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Update estate-rollout skill with mandatory smoke test gate before migration | Completed during this session (skill updated in-place) | Done |
| Corrective rollout: replace `permissions: {}` with `permissions:\n  contents: read` across all repos | Completed by lucos-system-administrator | Done |
| Convention checker updated to require `permissions:\n  contents: read` | lucas42/lucos_repos#178 | In progress (pending merge) |
| Close ~39 spurious audit issues opened by the premature convention rollout | Completed by lucos-issue-manager | Done |
| Add smoke test coverage for `code-reviewer-auto-merge` reusable workflow in `lucas42/.github` (currently only `dependabot-auto-merge` is tested) | lucas42/lucos#58 | Open |
| Estate rollout batching: 60s inter-batch gap insufficient to prevent avalon load spikes | lucas42/lucos#59 | Open |

---

## Sensitive Findings

[x] No — nothing in this report has been redacted.
