# ADR-0014: Migration-coupled deploys — process, not machinery

**Date:** 2026-07-02
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos_aithne/issues/261

## Context

lucos deploys automatically on merge to `main`. For the overwhelming majority
of changes this is exactly what we want: merge, build, deploy, done. It becomes
dangerous only for a specific class of change — a **breaking on-disk migration
that the newly-deployed image cannot perform for itself**.

On 2026-06-30 this class of change caused an estate-wide authentication outage.
`lucos_aithne`#260 changed how the signing-key encryption key (`SIGNING_KEK`)
was derived and *also* rotated to a fresh KEK value in the same release. The new
image could neither decrypt the existing signing key nor re-wrap it (the fresh
value is out-of-band operator material the running image cannot supply itself),
so it `log.Fatal`'d and crash-looped for ~46 minutes. New login and token
issuance were down across the estate. Full write-up:
[`docs/incidents/2026-06-30-aithne-kek-migration-deploy-race.md`](../incidents/2026-06-30-aithne-kek-migration-deploy-race.md).

The structural cause is that **auto-merge plus auto-deploy closes to zero the
human-occupied window** in which an operator would have run the one-time
migration before the new image served. The documented procedure tacitly assumed
that window existed; it does not.

Two families of remedy were explored on `lucos_aithne`#261:

- **Deploy-side machinery** — mark a release as "requires migration X" and hold
  the auto-deploy until an operator has run it; and/or add
  rollback-to-previous-image to the `lucos/deploy` orb when the readiness wait
  fails (the orb already health-gates via `docker compose up -d --wait
  --wait-timeout 180` over three attempts, but does not revert on failure).
- **Process** — rehearse the migration in a development environment first, and
  do not approve a migration-coupled PR for deployment until every manual step
  has a named owner who has confirmed they are ready to perform it.

On 2026-07-02 lucas42 gave the direction: **do not over-complicate the automated
deployment path.** The leverage is on the *approval* side, not the deploy
machinery side. This ADR records that decision and states the resulting
convention.

## Decision

Handle migration-coupled releases with a **process convention**, not new deploy
automation.

### Preference: make the migration need no manual step at all

The cheapest way to satisfy this convention is to have nothing to coordinate.
Where it is achievable, design a breaking on-disk migration to be
**self-contained and dual-read**, so the new image migrates the data forward at
startup with zero operator action — exactly as `lucos_aithne`'s existing
`MigrateSigningKeyEncryption` already does (try the current format; fall back to
reading the legacy format; re-wrap in place). In particular, **decouple a value
rotation from a format/derivation change**: ship the format change first as a
self-migrating deploy, then rotate the value online afterwards (e.g. aithne's
`--rekey`) once the new code is serving. A migration written this way is "fully
automated" and the two gates below simply do not apply to it.

The 2026-06-30 change did **not** have to be operator-dependent; it became so
only because it bundled a fresh-value rotation into a format change. Unbundling
them removes the manual step, and with it the race.

### When a manual step is unavoidable, two gates apply — both before approval

Some migrations genuinely cannot be made self-contained (they require out-of-band
operator material — a fresh secret, a decision, a value the running image cannot
derive). For those:

1. **Rehearse end-to-end in development first.** The full migration path —
   including every manual step, run exactly as it will be run in production — is
   exercised against a development environment before the PR is approved. Agents
   have full permissions in `development`, so there is no reason not to. A
   migration that has not been rehearsed end-to-end is not ready for approval.

2. **Gate approval on owner readiness.** Before a migration-coupled PR is
   approved for deployment, confirm that **every manual step has an assigned
   owner** (an agent and/or lucas42) who has **confirmed they are ready** to
   perform it. If any manual step lacks a confirmed-ready owner, **hold off on
   approval** until it does. Approval is the coupling point: because deploy
   follows merge automatically, the readiness of the humans/agents in the loop
   must be established *before* the merge, not assumed to exist after it.

### What is explicitly not built

- **No bespoke "release requires migration X" deploy-gate.** It adds standing
  complexity to the estate deploy path for a rare, now-mostly-avoidable event.
- **No orb rollback-on-unhealthy at this time.** It is not adopted here (see
  Alternatives); the convention above is the agreed remedy.

## Consequences

### Positive

- **No new standing complexity in the deploy path.** The busiest, most
  safety-critical shared component — the deploy orb, exercised by every service
  on every merge — is left alone. The control lives in the approval step, which
  is where a human/agent judgement already happens.
- **Addresses the real cause.** The 2026-06-30 outage was a *coordination*
  failure (nobody was lined up to run the migration at the moment it was
  needed), not a lack of tooling. Rehearsal + a readiness gate target that
  directly.
- **Rehearsal catches the friction too.** The recovery was slowed by stale
  procedure docs, a `PORT`-before-dispatch quirk, and a double-quoted-`.env`
  KEK trap (see the incident report). An end-to-end dev rehearsal surfaces all
  of these *before* production, when there is no outage clock running.
- **Scales down to zero.** A fully-automated/dual-read migration triggers
  neither gate, so the common case carries no process overhead.

### Negative

- **Enforced by discipline, not by the gate.** Nothing structural *stops* a
  migration-coupled PR from being approved without a rehearsal or without
  confirmed owners; it relies on the approver (agent or lucas42) applying the
  convention. This is the deliberate trade for not building machinery. The
  deferred follow-up wires it into the PR-approval flow so it is prompted for,
  not merely remembered.
- **"Migration-coupled" is a human judgement.** Whether a given PR needs an
  on-disk migration before it can serve is not detected automatically; the
  author and approver must recognise it. Self-contained migrations reduce how
  often the judgement matters, but do not remove it.
- **No automatic recovery if a bad deploy still slips through.** Because the orb
  still does not roll back, a migration-coupled release that reaches production
  without its migration would still crash-loop until manual intervention — the
  convention lowers the *probability* of that, not its blast radius. Fail-soft
  startup (`lucos_aithne`#262) shortens it but does not prevent it.

## Alternatives considered

- **Bespoke deploy-gate machinery** ("this release requires migration X; hold
  the deploy until it has run"). Rejected: it introduces a standing coupling
  between the release metadata and the deploy pipeline across the estate, for an
  event that is rare and, with the self-contained-migration preference above,
  mostly avoidable. The cost is paid on every deploy; the benefit accrues to a
  handful.

- **Rollback-to-previous-image in the `lucos/deploy` orb.** The orb already
  detects a failed deploy (the readiness wait fails the CI job); it simply does
  not revert. Adding "on readiness-wait failure, redeploy the previous image
  tag" would have bounded the 2026-06-30 outage to a brief blip and would guard
  *every* bad deploy, not just migration races. It is **not adopted here**, per
  the direction to keep the automated deployment path simple. Two honest caveats
  informed leaving it out: (a) on a single host with a fixed port it is
  rollback-on-unhealthy (a brief blip), not zero-downtime blue-green; and (b) it
  is only unconditionally safe when the failed deploy has not already written
  on-disk state the rolled-back old code cannot read — which is precisely why
  the self-contained-migration preference asks for forward- **and**
  backward-compatibility across the deploy window (standard expand/contract).
  Recorded here rather than adopted so the reasoning survives if the estate later
  chooses to revisit it as a general safety net independent of migrations.

- **Fail-soft startup** (a clean, specific halt — "signing key undecryptable
  under current `SIGNING_KEK` — run `--migrate-kek`" — instead of a crash-loop on
  "database may be corrupt"). Worth doing, but it is **MTTR, not prevention**:
  the new image still cannot serve; it just fails fast and legibly. Tracked
  separately as `lucos_aithne`#262, out of scope for this ADR.

## Deferred work

This ADR states a convention; for it to fire at the moment of action rather than
live only in this document, it must be surfaced in the PR-approval flow:

- **Wire the two gates into the approval step** so a migration-coupled PR is not
  approved (by `lucos-code-reviewer` or in triage) without (a) evidence of an
  end-to-end dev rehearsal and (b) a confirmed-ready owner for each manual step.
  A follow-up issue tracks the instruction change; it will be raised against this
  ADR before it is reported complete.
