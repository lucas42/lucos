# ADR-0011: Consumer tests must exercise the real interface of internal shared libraries

**Date:** 2026-06-06
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos/issues/219

## Context

On 2026-06-05, `lucos_loganne_pythonclient` released v2.0.0, which made `level` a required argument of `updateLoganne()`:

```python
# v1
def updateLoganne(type, humanReadable, url=None, **extra_data): ...
# v2
def updateLoganne(type, humanReadable, level, url=None, **extra_data): ...
```

This was a **deliberate, legitimate** breaking change. The library's SemVer major version is the channel through which it signals a breaking interface change, while the loganne *server* deliberately keeps `level` optional (defaulting to `routine`) per [`lucos_loganne` ADR-0001](https://github.com/lucas42/lucos_loganne/blob/main/docs/adr/0001-event-level-field.md). The contract is clean: a client on v1 of the library need not specify a level; a client on v2 must. Versioning the server's wire API to enforce this would have been substantial work for no real gain — the library's existing version number already provides the boundary.

The break itself is not what failed. What failed is that the breaking change reached production **silently**:

- By 04:15 UTC on 2026-06-06 the `lucos_arachne` scheduled ingestor was crashing on every run with `updateLoganne() missing 1 required positional argument: 'level'`, and monitoring went red (lucas42/lucos_arachne#608, fixed by #609).
- `lucos_photos` CI began failing on fresh dependency installs for the same reason.

The proximate enabler was that **consumers mock the shared library in their tests**. A test that patches `updateLoganne` with a bare mock keeps the *old* signature regardless of what the installed library actually does, so the test suite passed against v2, the Dependabot bump merged green, and the breakage only surfaced at runtime in production. The estate's safety net — CI gating dependency bumps — was looking at a mock, not the real interface, so it caught nothing.

Two narrower fixes were considered and rejected before settling on the decision below (both are recorded under *Alternatives considered*): reverting the client to make `level` optional again (lucas42/lucos_loganne_pythonclient#48, closed), and capping internal shared-library dependencies below the next major (lucas42/lucos_repos#409, closed).

## Decision

**Any repository that depends on an internal lucos shared client library MUST have at least one test that exercises the *real* library, driven against a *stubbed HTTP transport*** (e.g. `responses` or `requests-mock` for Python `requests`-based clients), covering the library's emit/call path.

Mocking the shared-library function itself — whether with a bare `Mock` or with an `autospec` mock — does **not** satisfy this requirement. The real library object must be the thing under test; only the outbound network call it makes is stubbed.

This works because the internal clients are thin HTTP wrappers: `updateLoganne()` validates its arguments and `POST`s a JSON payload to `LOGANNE_ENDPOINT`. Letting the real function run and intercepting only the HTTP call means the test goes through the library's actual signature and actual validation, so a breaking change to either fails the test.

Driving the real client this way catches **both** classes of breaking change, which is the point:

- **Arity breaks** — a newly-required argument (e.g. `level`) makes the real client raise `TypeError` when a caller omits it. This is the exact failure that took down the arachne ingestor.
- **Value / semantic breaks** — for example, `lucos_photos`' `api/app/services.py` calls `updateLoganne(event_type, human_readable, url)` positionally. Under v2 that binds the URL string into the new `level` slot, which the real client rejects via its `VALID_LEVELS` check with a `ValueError`. A signature-only check would pass this call — it is arity-valid — so **autospec is not sufficient**: autospec validates the shape of the call, not the values flowing through it, and this break lives in the values.

**Where a consumer currently has no test around its emit path at all** — a pure cron script with no test coverage of the call is the most likely vector by which the arachne break reached production untested — a test MUST be **added**. The requirement is not merely "stop mocking"; it is "exercise the real interface", and you cannot exercise what you do not test.

### Scope of "internal shared client library"

This applies to the reusable client packages produced by one repo and consumed by others as a code dependency — currently `lucos_loganne_pythonclient` and `lucos_schedule_tracker_pythonclient`, and any future client of the same shape. It is about *code* dependencies whose interface can drift under a version bump. It does not extend to plain HTTP integrations against another service's REST API (those have no installed interface to drift) nor to UI components like `lucos_navbar`.

## Alternatives considered

1. **Revert the client to make `level` optional** (lucas42/lucos_loganne_pythonclient#48). Rejected: the mandatory-via-major contract is a legitimate design choice, not a defect. Reverting it would treat a deliberate, correctly-signalled breaking change as a mistake, and would not address the reason the break reached production — the testing gap would remain for the next breaking change of any internal client.

2. **Cap internal shared-library dependencies below the next major** (e.g. `>=X.Y,<(X+1)`; lucas42/lucos_repos#409). Rejected: *not all breaking changes are breaking for a given consumer*. A cap forces every consumer to hand-bump for every major, including majors that change a function it never calls or an argument it already passes — friction with no safety the consumer actually needs. A real-interface test distinguishes "breaking for me" from "breaking in general" automatically: an actually-breaking bump fails that consumer's CI and stays unmerged, while a benign major passes and merges. That is strictly more discriminating than a blanket cap.

3. **`autospec` mocks** (`unittest.mock.create_autospec` / `patch(..., autospec=True)`). Better than a bare mock — an autospec mock built from the installed library raises on an arity break, so it would have caught the missing-`level` case. Rejected as the *mandated* bar because it misses value/semantic breaks (the `services.py` positional-misbind above passes autospec but fails the real client). Permitted only as a fallback where driving the real client against a stubbed transport is genuinely impractical, and even then it must be built from the installed library, never hand-specced.

4. **Library-provided spec'd fixture/fake.** The library could ship a test double (a faithful fake, or a pytest fixture) that performs the same validation as the real client, and consumers would use it instead of rolling their own mock. This is more robust than autospec and is the *only* option that is cleanly auditable — `lucos_repos` could check that consumers-with-tests import the provided fixture, whereas "is this mock autospec'd / is the transport stubbed" is not reliably detectable by static analysis. Not chosen as the mandate because it is heavier (the library takes on a test-helper surface to maintain) and couples every consumer to a library-specific test utility. A library MAY still offer such a fixture; doing so satisfies this ADR, since the fixture exercises the real interface.

## Consequences

### Positive

- **A breaking dependency bump fails the consumer's CI** — on the Dependabot PR *and* on any fresh-install build — so it cannot merge or deploy green. The safety net that was looking at a mock now looks at the real interface.
- **No cap friction.** Consumers are not forced to manually chase majors that do not affect them; only genuinely-breaking bumps stop the line.
- **Tests assert real behaviour.** Exercising the real client also covers its validation and payload shape, not a mock's possibly-stale assumptions about them — a test-quality improvement beyond the immediate signature concern.
- **Mechanism over policy.** The guarantee comes from code that runs in CI, not from a convention people must remember to honour on every version bump.

### Negative

- **Per-consumer test work to adopt**, and consumers that have no emit-path test must add one. This is real effort, tracked as the per-consumer follow-ups below.
- **Tests are marginally heavier** than a bare mock — a stubbed transport plus the real call rather than a one-line patch. Negligible in practice, and the fidelity is the entire point.
- **Not cleanly machine-auditable.** Unlike a version cap (which `lucos_repos` could check directly), "this test drives the real client against a stubbed transport" resists reliable static detection. Enforcement therefore rests on code review and on this ADR, unless a consumer adopts a library-provided fixture (alternative 4), whose *presence* is auditable. This is an honest limitation of the chosen approach: it is the more correct guard, but the weaker-to-enforce one.
- **Residual gap: untested paths stay unprotected.** The guarantee only covers emit paths that have a test. A code path with no test is caught by neither this ADR's mechanism nor by caps; the only remedy is adding the test, which is why "add one where missing" is part of the decision rather than an afterthought.

## Follow-up actions

To be filed/triaged as separate issues (the coordinator owns labels, priority, and board placement). The per-consumer migrations were filed alongside the discussion on lucas42/lucos#219:

- **`lucos_photos`** — migrate the three `updateLoganne` calls to pass `level` (including the `services.py` positional call → explicit keyword arguments) and add a real-transport emit test (lucas42/lucos_photos#420). CI is currently red; this is the deploy-gated one.
- **`lucos_eolas`** — migrate the three calls and add a real-transport emit test (lucas42/lucos_eolas#296).
- **`lucos_scheduled_scripts`** — pass `level` in the test script and verify the real installed client against a stubbed transport on build (lucas42/lucos_scheduled_scripts#42).
- **`lucos_backups`** (low — pinned to v1, not currently broken) — migrate the two calls and add a real-transport emit test when it adopts v2 (lucas42/lucos_backups#299).
- **`lucos_media_weightings`** (low — pinned to v1, already passes `level`) — add a real-transport emit test when it adopts v2 (lucas42/lucos_media_weightings#246).
- **`lucos_arachne`** — the compact.py coverage gap (lucas42/lucos_arachne#610) should use the real-transport pattern, not a mock.

Adopting this convention into [`docs/engineering-patterns.md`](../engineering-patterns.md) (which documents reality as it stands, not aspirations) is deferred until the consumers above have been migrated — at which point the pattern is reality and belongs there.
