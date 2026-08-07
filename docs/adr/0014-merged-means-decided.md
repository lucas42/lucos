# ADR-0014: Merged means decided — ADRs carry no `Status` field

**Date:** 2026-08-07
**Discussion:** [lucas42/lucos_worlds#71](https://github.com/lucas42/lucos_worlds/issues/71)

## Context

The lucos estate holds **63 ADRs across 18 active repositories** (measured 2026-08-07
by enumerating every `docs/adr/*.md` path in each non-archived repository through the
GitHub trees API; the archived `lucos_scheduled_scripts` holds one more, out of scope).
Every one of the 63 carries a `Status:` line: **45 read `Accepted`, 18 read `Proposed`,
none omit it.**

Nothing in the estate reads that field, or any part of an ADR:

- **No tooling.** `lucos_repos` audits 27 conventions and none of them look at ADRs.
  There is no template, no linter and no CI check. Every reference to `docs/adr` in
  code or CI — `lucos_aithne/main.go`, `lucos_firewall/main.go`,
  `lucos_contacts`' relationship-closure audit command, `lucos_repos`' CodeQL
  convention message, the schedule-tracker clients — is a prose comment pointing a
  human at a document.
- **No agent instruction.** The architect persona specifies "Context, Decision,
  Consequences" and has never mentioned a `Status` field. It has been propagated
  purely by copying whatever the neighbouring file did.

That last point is visible in the data. The field is universal, and no two repos
agree on how to write it:

```
most repos:                 **Status:** Accepted    (53 files — colon inside the bold)
lucos_worlds, lucos_contacts:  - **Status:** Accepted  ( 6 files — list bullet, colon inside)
lucos_monitoring:              - **Status**: Accepted  ( 4 files — list bullet, colon outside)
```

The same fragmentation runs through the rest of the format. Three title styles
(`# ADR-0001:` ×55, `# ADR 001:` ×2, `# 1.` ×6); two filename numbering widths
(4-digit everywhere except `lucos_media_metadata_api`, which uses 3); **five different
names for the field that links back to the deciding discussion** — `Discussion`,
`Issue`, `Related`, `Related issues`, `Discussed in` — plus repos carrying no such
field at all. Only three things are genuinely uniform: ISO-8601 dates, and the
sections `## Context`, `## Decision` and `## Consequences`, which all 63 files carry.

There is **no document anywhere in the estate defining the ADR format**. The standard
is convention-by-imitation, which is exactly the mechanism by which an undefined field
became universal and then drifted into three spellings without anyone noticing.

### What the field actually encodes

`Status` conflates two independent questions — *has this decision been agreed?* and
*has it been implemented?* — into one word, and in practice it has been used for both:

- **`lucos_worlds` ADR-0001 and ADR-0002** read `Proposed` for a month while the
  system they describe was deployed and running in production. The decisions were
  agreed and the implementation had shipped; the field was simply stale. This is what
  raised [lucas42/lucos_worlds#71](https://github.com/lucas42/lucos_worlds/issues/71).
- **`lucos` ADR-0013** reads `Proposed`, and here the field is tracking
  *implementation*: lucas42 approved [lucas42/lucos#237](https://github.com/lucas42/lucos/pull/237)
  at 01:13:54Z on 2026-06-11, fifteen minutes before it merged, so the decision was
  agreed. What has not happened is the build — the `additionalReviewers` field the
  Decision specifies is absent from the live `lucos_configy` API, and both
  implementation tickets ([lucas42/.github#70](https://github.com/lucas42/.github/issues/70),
  [lucas42/lucos_claude_config#114](https://github.com/lucas42/lucos_claude_config/issues/114))
  are still open.

Both readings are defensible from the word alone, which is the problem: a reader
cannot tell which sense a given `Status` line is being used in, nor whether the value
is maintained or stale.

### The failure that prompted this

On 2026-08-06 both `team-lead` and the architect independently cited `lucos` ADR-0013
as the live auto-merge policy, having read its header and its Context section. The
Context accurately describes today's gate — because that is the status quo the ADR
argues *against*. The `Status: Proposed` line did not prevent the misreading and
arguably contributed to it: a header invites a reader to treat it as the answer
instead of checking whether the Decision shipped.

## Decision

### 1. Merged means decided

**An ADR on a repository's default branch records a decision that has been made.**
An ADR that is not merged — an open or draft PR — is a proposal, not a decision
record. The "D" is Decision; an un-agreed proposal is a design document.

The corollary is binding: **do not merge an ADR PR until the decision it records has
been agreed.** This is already the practice — ADRs ship as draft PRs pending lucas42's
sign-off — and this ADR makes it the rule rather than the habit.

### 2. ADRs carry no `Status` field

The field is removed from existing ADRs and is not written into new ones. The
invariant in §1 carries the same information, cannot go stale, and is enforced by
version control rather than by hand.

### 3. Supersession is a pointer, not a status word

When an ADR — or part of one — is superseded, the **superseding ADR's own PR** adds
the pointer:

- **Whole ADR superseded:** a `**Superseded by:**` header field naming the superseding
  ADR.
- **Part superseded:** a note at the top of the affected section naming the superseding
  ADR and what it replaces. This is the common case and the one a status word could
  never express — see `lucos_worlds` ADR-0001 §1, superseded by `lucos_worlds` ADR-0004
  while the rest of ADR-0001 still stands.

A pointer is strictly more informative than the word `Superseded`: it names *what*
replaced the decision and *which part*.

### 4. Implementation tracking lives in GitHub issues

An ADR's `## Deferred work` section links the issues that track its implementation.
Whether an ADR has shipped is answered by those issues and by the running system, never
by the document. This is lucas42's framing on
[lucas42/lucos_worlds#71](https://github.com/lucas42/lucos_worlds/issues/71): *"The ADR
is about the decision, not tracking the implementation."*

### 5. The ADR format is written down

The normative format lives at [`docs/adr/README.md`](README.md) in this repository and
applies to **every** lucos repository, not just this one. It fixes filename and
numbering, the title line, the permitted header fields, the required and recommended
sections, and cross-repo reference qualification.

That document is a **living standard**: cosmetic clarifications to it do not need a new
ADR. The decisions in §1–§4 above do — changing them means changing what an ADR *is*,
and that belongs in a numbered record.

## Consequences

### Positive

- **The invariant cannot go stale.** "Is this decided?" is answered by whether the file
  is on `main`, which no one has to remember to update. The failure mode that produced
  this ADR — a hand-maintained field reading `Proposed` for a month on a system running
  in production — is structurally removed, not merely corrected.
- **The two questions are separated.** "Agreed?" is answered by version control;
  "shipped?" is answered by the implementation issues and the running system. Neither
  can be mistaken for the other.
- **Supersession gets a better home**, carrying the target and the scope rather than a
  single word.
- **The format is defined for the first time**, so the next agent writing an ADR copies
  a standard rather than a neighbour.

### Negative

- **Migration cost: 63 files across 18 repositories.** Every existing ADR needs its
  `Status` line removed. The edit is mechanical, but it is 18 separate PRs across 18
  repos, each needing review. This is the honest price of the invariant and it is not
  small.
- **Four ADRs become decided by fiat.** Of the 18 currently reading `Proposed`, **14
  carry an explicit `lucas42` APPROVED review on the PR that introduced them** — their
  `Proposed` line was tracking implementation, not agreement, and dropping the field
  loses nothing. **Four merged on a bot approval alone**, with no human sign-off
  recorded on the PR:

  | ADR | Introducing PR | Merged |
  |---|---|---|
  | `lucos_repos` ADR-0005 (CodeQL policy by repo class) | lucas42/lucos_repos#315 | 2026-04-10 |
  | `lucos_claude_config` ADR-0003 (skill-based persona structure) | lucas42/lucos_claude_config#53 | 2026-05-08 |
  | `lucos` ADR-0009 (artist identity & membership federation) | lucas42/lucos#208 | 2026-06-01 |
  | `lucos_repos` ADR-0007 (generated convention catalogue) | lucas42/lucos_repos#437 | 2026-06-22 |

  Under §1 these become decided. That may well be correct — the absence of an approval
  review is **not** evidence that lucas42 disagreed, and several were discussed on their
  tickets — but it is an inference, not a record, and each deserves a look rather than a
  sweep. Tracked as deferred work below.
- **The `Superseded` signal can still be forgotten.** Writing the pointer is a manual
  act, just as maintaining `Status` was. The difference is the failure direction: a
  *missing* pointer costs a reader a search, whereas a *stale* status actively
  misleads them. That is an improvement, not an elimination.
- **The invariant is enforced by convention, not by a gate.** Nothing structurally
  prevents an un-agreed ADR being merged — on an unsupervised repo the auto-merge
  workflow fires within seconds of the code-reviewer bot's approval. The draft-PR
  convention is the only control, and the four ADRs above are its measured leak rate
  (4 of 63) over four months. §1 raises the stakes of that leak, because merging now
  *means* something.
- **A signal that failed is being removed rather than fixed.** `Proposed` on ADR-0013
  is what should have stopped two agents misreading it as live policy. The
  counter-argument deserves stating plainly: removing a safeguard that did not work is
  not automatically an improvement. It is defensible here because the field failed for
  a structural reason — it duplicated a fact nothing enforced, so it could drift, and a
  reader had no way to tell a maintained value from a stale one. Replacing it with an
  invariant version control enforces is replacing an unreliable control with a reliable
  one. The real lesson from that misreading is orthogonal and holds regardless: **read
  the Decision and check whether it shipped; a header is never evidence.**

## Alternatives considered

- **Keep `Status` and fix only the two `lucos_worlds` ADRs** — the original scope of
  lucas42/lucos_worlds#71. Cheapest option, and it leaves the drift mechanism in place
  in all 18 repos. We would be back here.
- **Redefine `Status` to track implementation explicitly** (e.g.
  `Proposed`/`Accepted`/`Implemented`). Rejected: it makes the document a hand-maintained
  mirror of issue state, which is a second source of truth for something GitHub already
  answers, and it drifts for exactly the same reason the current field does.
- **Derive the status automatically from linked issues** and render it. Rejected as
  disproportionate: it is machinery — a parser, a renderer, a CI job — built to solve a
  problem that disappears entirely if the field is simply removed.
- **Enforce the format with a `lucos_repos` convention.** Tempting, since undefined-plus-
  unenforced is what allowed three spellings to coexist. Rejected for now: nothing reads
  ADRs, the drift that actually caused harm is removed by §2 rather than by enforcement,
  and the residual variance (header field naming, title style) is cosmetic. A convention
  would mean a new Go check plus tests plus an estate rollout to make 18 repos pass, to
  police document cosmetics. Worth revisiting only if the `Status` field creeps back in
  by imitation after removal — which is the one regression an audit check would
  genuinely catch.

## Deferred work

- **Remove the `Status` field from all 63 existing ADRs** across the 18 repositories
  that hold them. Mechanical; one PR per repo.
- **Review the four ADRs merged without a recorded human approval** (table above) and
  confirm each is a decision we intend to be bound by, or supersede it.
- **Update the agent instructions in `lucos_claude_config`** so the architect persona
  and the code-reviewer workflow reference this standard rather than perpetuating the
  copy-your-neighbour pattern — including removing the assumption that a `Status` flip
  is a thing that happens to an ADR.

Follow-up issues for each are raised against this ADR before it is reported complete.
