# Architecture Decision Records — the lucos standard

This document defines the ADR format for **every** lucos repository, not just this one.
It is normative. The decisions behind it are recorded in
[ADR-0014](0014-merged-means-decided.md); this document holds the detail, and may be
clarified without a new ADR so long as those decisions are unchanged.

## What an ADR is

A record of a decision that has been made, and the reasoning that produced it — written
so that someone reading it in three years can tell not just *what* was decided but *why*,
and what was traded away.

It is **not** a design proposal, a specification, or an implementation tracker.

## Merged means decided

**An ADR on a repository's default branch is a decision that has been agreed.** An
unmerged ADR — an open or draft PR — is a proposal.

This is the whole of the ADR's status. There is **no `Status:` field**; do not add one.
Whether the decision has been *implemented* is a separate question, answered by the
issues listed under `## Deferred work` and by the running system — never by the document.

The corollary binds the author: **do not merge an ADR PR until the decision is agreed.**
On unsupervised repos that means opening it as a draft, getting lucas42's sign-off, and
only then marking it ready.

## Which repository it belongs in

| The decision… | ADR lives in |
|---|---|
| establishes a **brand-new system** | that system's own repo, as `ADR-0001` |
| changes **one existing system** | that system's repo |
| spans **more than one existing system** | `lucas42/lucos` |

A new-system founding ADR that also defines a cross-system contract is still a founding
ADR — it belongs in the new system's repo and references the cross-system implications
from there.

## File and numbering

- Path: `docs/adr/` in the owning repository.
- Filename: `NNNN-kebab-case-title.md` — four digits, zero-padded, no gaps.
- Numbering is **per-repository** and starts at `0001`. Numbers are never reused, even
  if an ADR is superseded.

Because numbering is per-repo, `lucos_arachne` ADR-0004 and `lucos` ADR-0004 are
different documents. **Always qualify an ADR reference made from outside its home repo**
— write "`lucos_arachne` ADR-0004", never a bare `ADR-0004`.

## Structure

```markdown
# ADR-NNNN: Title in sentence case

**Date:** YYYY-MM-DD
**Discussion:** [lucas42/repo#N](https://github.com/lucas42/repo/issues/N)

## Context
## Decision
## Consequences
## Alternatives considered
## Deferred work
```

### Title line

`# ADR-NNNN: Title` — matching the filename's number. The title names the decision, not
the topic: "Merged means decided", not "ADR status".

### Header fields

Immediately after the title, one per line, in `**Field:** value` form (colon inside the
bold). Only `Date` and `Discussion` are required:

| Field | Meaning |
|---|---|
| `**Date:**` | ISO-8601 date the decision was made. **Required.** |
| `**Discussion:**` | Link to the issue or PR where it was decided. **Required.** |
| `**Superseded by:**` | Set when a later ADR replaces this one wholesale. |
| `**Supersedes:**` / `**Amends:**` / `**Extends:**` | Relationship to an earlier ADR. |
| `**Incident:**` | The incident report that forced the decision, where there was one. |

Do not invent further fields without adding them here — six different spellings of
"Discussion" is how the estate got into the state ADR-0014 describes.

### Sections

- **`## Context`** *(required)* — the situation that forced a decision, including the
  constraints and the evidence. Note that Context describes the status quo the ADR is
  reasoning *about*, which may be the very thing it argues against; write it so a reader
  cannot mistake it for the decision.
- **`## Decision`** *(required)* — what was decided, stated in the active voice and
  precisely enough to be implemented from. Number the clauses if there is more than one.
- **`## Consequences`** *(required)* — split into positive and negative. **State the
  trade-offs honestly.** An ADR with no negative consequences is not a decision; it is
  an advertisement. Costs, migration burden and residual risk belong here, quantified
  where they can be.
- **`## Alternatives considered`** *(strongly recommended)* — what else was weighed and
  why it lost. This is usually the section a future reader needs most, because it is the
  only record of the options that were *not* taken.
- **`## Deferred work`** *(required when the ADR defers anything)* — every piece of work
  the ADR explicitly leaves for later, each with a tracked GitHub issue. An ADR is not
  complete until they exist.

Further sections are fine where the decision needs them.

## Superseding and amending

An ADR is a record of what was decided *at a point in time*. Never rewrite a decision in
place — that erases both that the original choice was made and why it changed.

- **A new consideration, or a reversal, gets its own numbered ADR.** The new ADR's PR
  also adds the pointer to the old one: a `**Superseded by:**` header field if the whole
  ADR is replaced, or a note at the top of the affected section if only part is. The
  original text stays intact.
- **An amendment** is for additive detail *within the scope of the decision the ADR
  already made* — a clarification, not a change of mind. Add it as a clearly marked
  section at the end, dated, leaving the `## Decision` unchanged.

Sharing a *mechanism* with an existing ADR is not a reason to fold a new decision into
it; the shared mechanism is a consequence of both decisions, not evidence they are one.

## Citing an ADR

An ADR's presence on `main` tells you the decision was agreed. It tells you **nothing**
about whether it has been built. Before citing an ADR as the source of an obligation,
read its `## Decision` and check the implementation actually shipped — does the field,
route or behaviour exist; are its `## Deferred work` issues closed. A mechanism you
recognise in the `## Context` section is not evidence of anything: that is the world the
ADR is arguing about, and quite possibly arguing against.
