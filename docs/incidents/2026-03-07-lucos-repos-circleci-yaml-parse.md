# Incident: lucos_repos CircleCI convention checks fail across all repos

**Date:** 2026-03-07
**Duration:** ~24 minutes (13:30 UTC to 13:54 UTC)
**Severity:** Partial degradation
**Services affected:** lucos_repos convention check system
**Detected by:** lucos-site-reliability ops check / issue raised by agent immediately after PR #75 merged

Source issue: https://github.com/lucas42/lucos_repos/issues/80

---

## Summary

When PR #75 (replacing the legacy `has-circleci-config` check with five nuanced per-type CircleCI conventions) was merged, a YAML parsing bug caused virtually every CircleCI convention check to fail immediately for all repos. The `Workflows` field in the parser was typed as `map[string]ciWorkflow`, but `yaml.v3` cannot unmarshal the scalar `version: 2` key that appears inside the `workflows` block of every standard CircleCI config. As a result, `parseCIConfig` returned an error for nearly every repo, and the convention runner opened spurious "convention failing" issues across 16+ repos. The bug was identified within 19 minutes of the causative PR merging, and fixed in PR #81 which was merged 5 minutes after the issue was raised.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 13:30 | PR #75 merged — five nuanced CircleCI conventions deployed |
| ~13:30–13:49 | Convention runner begins processing repos; `parseCIConfig` fails on `version: 2` for every standard CircleCI config; spurious "convention failing" issues opened across 16+ repos |
| 13:49 | Issue #80 raised documenting root cause and proposing fix options |
| 13:50 | lucos-developer begins work on PR #81 |
| 13:54 | PR #81 merged — YAML parser fixed; spurious issues cleaned up (16 closed as part of the fix) |

---

## Root Cause

The five new CircleCI conventions in PR #75 all depend on `parseCIConfig` in `conventions/circleci_helpers.go` to deserialise `.circleci/config.yml` files. The parser declared the `Workflows` section as `map[string]ciWorkflow` — a Go map whose values are structs. However, virtually every real CircleCI config file includes the line `workflows.version: 2` inside the `workflows` block. This is a scalar integer value alongside the workflow map entries. The `yaml.v3` library cannot unmarshal an integer into a `ciWorkflow` struct and returns an unmarshal error immediately. Since this affected the root config object, all five CircleCI conventions reported a parse error rather than actually checking the config, which the convention runner interpreted as a failing convention and opened issues accordingly.

The bug existed in the code as shipped in PR #75 but was not caught by tests because the test fixtures did not include `workflows.version: 2`.

---

## What Was Tried That Didn't Work

No dead ends were recorded from the issue or PR discussion — the root cause was identified correctly on the first attempt and the fix worked immediately.

---

## Resolution

PR #81 implemented `UnmarshalYAML` on `circleCIConfig`, replacing the direct yaml.v3 decoding of the `Workflows` field with a two-pass decode:

1. An alias type captures the raw `workflows` node as `yaml.Node`.
2. The method walks the mapping node's content manually, skipping any entry whose value is a scalar (not a mapping) — silently ignoring `version: 2` and any similar scalar entries.
3. Only mapping-valued entries are decoded into `ciWorkflow`.

The `Workflows` field was tagged `yaml:"-"` so the standard decoder never touches it. A new test `TestParseCIConfig_HandlesWorkflowsVersionKey` was added using a realistic config including `workflows.version: 2`.

As part of the fix deployment, 16 spurious "convention failing" issues across repos were closed.

---

## Contributing Factors

### Missing test coverage for real-world CircleCI config format

The test fixtures used to develop and review PR #75 did not include `workflows.version: 2`, which is ubiquitous in real CircleCI configs. Had even one test used a realistic config, the parser error would have been caught before merge.

### No staging or canary roll-out for convention checks

Convention checks run immediately on all repos after deployment. There is no mechanism to run a check against a representative sample of repos before enabling it globally, so any parsing error affects the entire estate simultaneously.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Fix YAML parsing of `workflows.version: 2` in CircleCI configs | lucos_repos#81 | Done |
| Surface convention `Detail` string in all audit issue bodies (so spurious issues are easier to triage) | lucos_repos#82 | Open |
| Consider adding a representative real-world fixture to CircleCI convention tests | No issue yet | Open |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[x] No — nothing in this report has been redacted.
