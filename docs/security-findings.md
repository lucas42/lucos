# Security Findings Workflow

This document explains how security findings are handled in lucos projects: when they go to the normal issue tracker, and when a private GitHub Security Advisory is used instead.

## Default: public GitHub issues

All lucos repos are public. The normal workflow for security findings — like everything else — runs through the public GitHub issue tracker, where agents can triage, route, and implement fixes in the standard pipeline.

Most security findings belong there. A finding that requires specific error conditions to trigger, that is only exploitable by someone who already has privileged access, or that represents a defence-in-depth gap rather than an immediate attack path, is not meaningfully made worse by being public. Keeping it in the issue tracker means it gets worked on efficiently.

## Exception: private GitHub Security Advisories

A small category of findings warrant a private advisory rather than a public issue. Use an advisory only if **both** of these are true:

1. An attacker with network access could exploit it **immediately**, without needing any other pre-existing access or insider knowledge
2. The finding is not yet fixed

If both conditions are met, creating a public issue would hand an attacker a ready-made exploitation guide. In that narrow case, use a private [GitHub Security Advisory](https://docs.github.com/en/code-security/security-advisories/working-with-repository-security-advisories/about-repository-security-advisories) on the affected repository.

If only one condition is met — for example, the finding is immediately exploitable but has already been fixed, or it requires existing access to trigger — it goes as a normal public issue.

**When in doubt, default to public.** The overhead of the advisory path is only justified for genuinely dangerous, unfixed findings.

## Advisory workflow

1. Create a draft advisory on the affected repository. Draft advisories are private — visible only to repo collaborators, not to GitHub or the public.
2. Optionally, open a public issue that references the advisory obliquely (e.g. "a security finding is being tracked privately and will be resolved shortly") so the work is visible in the normal pipeline without exposing details.
3. Implement the fix as normal — a PR referencing the issue or advisory.
4. After the fix is merged, update the advisory to reflect the resolution. Publishing it is optional; for a single-user project not in anyone's supply chain, leaving it as a closed draft is fine.

## Examples

| Scenario | Routing |
|---|---|
| Unauthenticated endpoint on a public-facing service allows arbitrary data deletion | Advisory (immediately exploitable, no prerequisites) |
| SMTP credentials may appear in internal logs under a specific error condition | Public issue (requires existing log access; not immediately externally exploitable) |
| A trusted service list, if edited maliciously, could expand an attacker's reach | Public issue (requires existing privileged access to trigger) |
| SQL injection in a login form, not yet patched | Advisory (immediately exploitable by any network-connected attacker) |
| Known vulnerable dependency, no active exploitation path identified | Public issue (conditional; goes through normal dependency update flow) |

## Related

- [GitHub Security Advisories documentation](https://docs.github.com/en/code-security/security-advisories)
- Agent instructions: `lucos-security` and `lucos-architect` in `lucos_claude_config`
