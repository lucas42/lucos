# ADR-0013: Data-driven auto-merge approval policy

**Date:** 2026-06-11
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos_configy/issues/224

## Context

Whether a pull request auto-merges across the lucos estate is decided in three
separate places today, only one of which is enforced by the merge gate itself:

1. **Supervision** — `lucos_configy` declares a per-repo `unsupervisedAgentCode`
   boolean. The reusable auto-merge workflow (`reusable-code-reviewer-auto-merge.yml`
   in `lucas42/.github`) reads it on each `pull_request_review` event and merges
   when:
   - `unsupervisedAgentCode == true` → the triggering approval is from
     `lucos-code-reviewer[bot]`, **or**
   - `unsupervisedAgentCode == false` → the triggering approval is from `lucas42`.
2. **Security-critical repos** — the rule that `lucos_firewall`, `lucos_creds`
   and `lucos_aithne` additionally require `lucos-security` review lives as a
   hardcoded list in `lucos-code-reviewer`'s *instructions*. It is enforced only
   by the agent withholding its own approval — **not** by the merge gate.
3. **Per-PR specialist judgement** — `lucos-code-reviewer` may decide a given PR
   is security- or reliability-relevant and request `lucos-security` /
   `lucos-site-reliability` review ad-hoc. This is a subjective call and is also
   enforced only by the agent.

Three problems follow from this arrangement:

- **The security-critical list is not data-driven and not enforced by the gate.**
  It is prose in an agent instruction file. If the agent misjudges, is offline,
  or the instruction drifts, nothing structural stops a security-critical repo's
  PR from merging without `lucos-security` having looked at it. A list of *which
  repos always need a given reviewer* is configuration, not judgement — it should
  live in `lucos_configy` and be enforced by the workflow.
- **Supervised repos merge on `lucas42` alone.** The current workflow checks only
  the *triggering* reviewer. On a supervised repo, `lucas42`'s approval merges the
  PR even if `lucos-code-reviewer[bot]` never reviewed it. lucas42 wants supervised
  repos to require **both** `lucos-code-reviewer` and `lucas42` — so the bot's
  systematic checks (CI, conventions, scope) always run before a human sign-off
  completes the merge.
- **No extension point for other always-required reviewers.** There is nowhere to
  declare, for example, that `lucos_monitoring` should always carry
  `lucos-site-reliability` review. Today that would mean editing agent instructions
  again.

What should **not** change: the *subjective* "is this particular PR
security-relevant?" call. That is genuinely a judgement (problem 3 above) and
cannot be reduced to per-repo config — it stays with `lucos-code-reviewer`.

This is an estate-wide governance decision (it spans the `lucos_configy` schema,
the `lucas42/.github` auto-merge workflow, and `lucos-code-reviewer`'s
instructions), so it is recorded as a `lucos` estate ADR rather than in any one
system's repo — directly analogous to `lucos` ADR-0007, an estate port policy
carried by `lucos_configy` fields and enforced by `lucos_firewall`.

## Decision

Make the required-approver set for auto-merge **data-driven from `lucos_configy`
and enforced by the auto-merge workflow**, in three parts.

### 1. `lucos_configy` gains a per-repo `additionalReviewers` field

A new optional field on the repository-bearing entity types (`System`,
`Component`, `Script` — mirroring where `unsupervisedAgentCode` already lives):

```yaml
lucos_firewall:
    hosts: [avalon, salvare, xwing]
    additionalReviewers:
        - lucos-security
```

- Type: list of **semantic reviewer names** (strings), e.g. `lucos-security`,
  `lucos-site-reliability`. **Not** GitHub logins or numeric IDs — see §2 on why
  the identity mapping belongs in the enforcement layer.
- Default: empty list `[]` (most repos require no additional reviewer).
- Serialisation: serialised with serde rename to camelCase (`additionalReviewers`)
  for consistency with `unsupervisedAgentCode`, and **always present** in the JSON
  response as an array (empty when unset), per configy's optional-field contract.
  The hand-written RDF/turtle serializer in `api/src/all.rs` must be updated in
  step with `api/src/data.rs` (the compiler does not catch omissions there — see
  configy's `api/CLAUDE.md`).

Initial population: `additionalReviewers: [lucos-security]` on `lucos_firewall`,
`lucos_creds` and `lucos_aithne` — replacing the hardcoded list in the agent
instructions. Whether `lucos_auth_scopes` (already supervised, so already
`lucas42`-gated) also warrants `lucos-security` is lucas42's call at population
time; it is a config decision, not a schema one.

### 2. The auto-merge workflow computes and enforces the full required set

On each `pull_request_review` (submitted) event the reusable workflow computes the
**required-approver set** for the repo:

- **always** `lucos-code-reviewer[bot]`;
- **plus** `lucas42` if `unsupervisedAgentCode == false`;
- **plus** every entry in `additionalReviewers`.

It then fetches **all** reviews on the PR and, for each required approver,
evaluates **the most recent review by that approver**. Auto-merge is enabled only
when the latest review from *every* required approver is `APPROVED`. This is
order-independent by construction: a later `REQUEST_CHANGES` or dismissal from any
required approver removes the approval; a re-approval restores it.

Identity verification: each required approver is matched on **both** GitHub login
**and** numeric account/app ID, exactly as the current workflow already does for
`lucos-code-reviewer[bot]` and `lucas42` — guarding against a renamed-account
spoof. The map from semantic name (`lucos-security`) → `(login, numeric id)` lives
**in the workflow**, the enforcement layer, not in `lucos_configy`. Two reasons:
the numeric IDs are an implementation detail of *this* enforcement mechanism (a
different consumer of the same config has no use for them), and concentrating the
security-sensitive identity facts in the workflow keeps configy a clean,
human-meaningful declaration.

**Fail closed.** Consistent with the workflow's existing supervision-lookup
handling, auto-merge must be **blocked with a loud error** (never a silent skip)
when: configy is unreachable or returns non-200; the `unsupervisedAgentCode` or
`additionalReviewers` field is absent; or an `additionalReviewers` entry cannot be
mapped to a verified identity. An unknown or unverifiable required reviewer must
hold the merge, not pass it.

### 3. `lucos-code-reviewer` instructions drop the static list, keep the judgement

The hardcoded "always-review repos" list (`lucos_firewall`, `lucos_creds`,
`lucos_aithne` → `lucos-security`) is removed from the agent's `review-pr`
workflow, because the merge gate now enforces it from configy. The agent retains:

- its standing role as a required approver (it is always in the set), so it still
  reviews every PR; and
- the **subjective** per-PR specialist-request judgement (problem 3) — when it
  judges a PR in a *non-listed* repo to be security/reliability-relevant, it
  requests that specialist and withholds its own approval pending their response.
  Because the agent is itself a required approver, withholding its approval blocks
  the merge. This dynamic path stays agent-enforced; only the *static, per-repo*
  requirement moves to config.

Dependabot PRs are unaffected: `reusable-dependabot-auto-merge.yml` is a separate
workflow that does not consult `unsupervisedAgentCode` or `additionalReviewers`.

## Consequences

### Positive

- The "which repos always need which reviewer" policy becomes a single,
  inspectable source of truth in `lucos_configy`, enforced by the merge gate
  rather than by agent goodwill. A security-critical repo cannot auto-merge
  without `lucos-security`'s approval even if the reviewing agent misjudges or is
  offline.
- Supervised repos now genuinely require both the bot's systematic checks and a
  human sign-off, in any order.
- Adding `lucos-site-reliability` to `lucos_monitoring` (or any reviewer to any
  repo) is a one-line config change with no agent-instruction edit and no
  branch-protection reconfiguration.
- The supervision determinant (`unsupervisedAgentCode`) keeps its single, existing
  meaning; `additionalReviewers` layers on top rather than restating it, so there
  is no second source of truth for supervision.

### Negative / accepted trade-offs

- **A supervised PR can no longer auto-merge on `lucas42` alone** — the bot must
  also have a current approval. If the reviewing agent is unavailable, a supervised
  PR will not auto-merge even with lucas42's approval. Mitigation: lucas42 can
  always merge manually through the GitHub UI; the workflow gates *auto*-merge, not
  human merge. This is the intended tightening, not a regression.
- **The workflow now does more on each review event** — a reviews-list fetch and
  per-approver evaluation rather than a single triggering-reviewer check. This is a
  handful of extra API calls per review; negligible, and it inherits the existing
  retry/fail-closed scaffolding.
- **Residual risk: dynamic specialist gating is only as strong as the agent.** For
  repos *not* in `additionalReviewers`, a PR that the agent *should* have flagged as
  security-relevant but didn't can still auto-merge (on an unsupervised repo, on the
  bot's own approval). The config enforces *static* per-repo reviewers; it cannot
  enforce a *subjective* per-PR call — lucas42 explicitly accepts this (configy#224:
  "subjective calls … will need to be made by an agent"). The lever, if a repo's PRs
  turn out to be *frequently* security-relevant, is to promote it to a static
  `additionalReviewers` entry.
- **A new semantic-name → identity mapping must be maintained in the workflow.**
  Adding a never-before-used reviewer requires both the configy entry *and* a
  one-time map addition in the workflow; until the map entry exists the workflow
  fails closed (blocks merge) for that repo. This is the safe failure direction but
  is a two-touch operation the first time a reviewer is introduced.

## Alternatives considered

- **A single unified `requiredReviewers` list per repo** enumerating *all* required
  approvers explicitly (`[lucos-code-reviewer, lucas42, lucos-security]`). Rejected:
  it restates the supervision fact (`lucas42 ⟺ unsupervisedAgentCode == false`) that
  `unsupervisedAgentCode` already carries, creating two sources of truth that can
  drift. Layering `additionalReviewers` on top of the existing field keeps
  supervision single-sourced.
- **GitHub branch protection / rulesets + CODEOWNERS** to require approvals
  natively. Rejected as the primary mechanism: branch-protection rules are
  per-repo repo-settings (sysadmin-only, not declarative in configy), so the
  required-reviewer policy would no longer be sourced from configy — splitting the
  source of truth and reintroducing drift across ~30 repos. Bot apps as code owners
  is also awkward (team/write-access requirements). A required *status check* set by
  this workflow and demanded by branch protection is noted as a possible future
  hardening (it would block a racing manual merge from bypassing the gate), but is
  not needed today: on the gated repos only `lucas42` and trusted bots can merge.
- **Storing GitHub numeric IDs in `lucos_configy`.** Rejected: they are an
  enforcement-mechanism detail with no value to other config consumers, and
  concentrating identity facts in the workflow keeps configy human-meaningful. See
  §2.

## Deferred work

This ADR defines the design; the implementation is tracked separately:

- **`lucos_configy`** — add the `additionalReviewers` field (`data.rs`, `all.rs`,
  README, tests) and populate `lucos_firewall`, `lucos_creds`, `lucos_aithne`.
- **`lucas42/.github`** — rewrite `reusable-code-reviewer-auto-merge.yml` to compute
  the required-approver set, enforce latest-review-per-approver, hold the
  name→identity map, and fail closed; extend the smoke-test suite accordingly.
- **`lucos_claude_config`** — remove the static always-review list from
  `lucos-code-reviewer`'s `review-pr` workflow, retaining the subjective
  specialist-request judgement.

Follow-up issues for each will be raised against this ADR before it is reported
complete.
