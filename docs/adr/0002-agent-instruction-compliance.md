# ADR-0002: Agent instruction compliance for structured task lists

**Date:** 2026-03-06
**Status:** Proposed

## Context

lucos uses AI agents (Claude Code personas) to perform recurring operational tasks. Each agent has a persona file (in `~/.claude/agents/`) that defines its identity, responsibilities, and — critically — structured task lists such as ops checks that must be executed completely.

On 2026-03-06, the lucos-site-reliability agent ran its ops checks routine. Its persona file (`lucos-site-reliability.md`, 546 lines) defines 6 checks. The agent ran checks 1-5 and stopped, silently skipping Check 6 (incident report coverage). When asked why, the agent stated "there is no Check 6 in my instructions" — which was false. A second agent asked to audit the file also failed to notice Check 6 existed. Both agents could see Check 6 clearly when asked to read the file directly.

This was not a file loading or truncation problem. The instructions were present and complete. Check 6 starts at line 199 — not even near the end of the 546-line file. The agents failed to attend to it during task execution, and confabulated when asked to explain the omission.

This failure mode — silent omission of instructions in long files, followed by confident denial — is an architectural reliability concern. It affects any persona with structured task lists embedded in long instruction files. Current persona file lengths range from 237 to 546 lines, with several exceeding 300.

## Decision

Adopt six practices to improve agent compliance with structured instructions. These are ordered by expected impact, and the first three are the most important.

### 1. Extract structured task lists into separate short files

Structured task lists (such as ops checks, review checklists, or triage procedures) must be extracted from persona files into dedicated files, kept under 200 lines each. The persona file references the task list file and instructs the agent to read it at the start of the relevant workflow.

For example, the SRE ops checks would move from `lucos-site-reliability.md` to a separate file (e.g. `~/.claude/agents/sre-ops-checks.md`). The persona file would contain:

```
## Ops Checks
When running ops checks, first read `~/.claude/agents/sre-ops-checks.md`
in full, then execute every check listed there.
```

**Rationale:** Language models process instructions with decreasing attention fidelity as document length increases. This is not a hard cutoff but a gradient — items deeper into a long document are progressively more likely to be missed. By reading the task list at execution time rather than relying on it being in the system prompt, the content enters the most recent context window where attention is strongest. Shorter, focused files also distribute attention more uniformly across their content.

### 2. Explicit count and completion manifest

Every structured task list must include an explicit item count at the top and require the agent to produce a completion manifest after execution.

The task list file should open with:

```
# SRE Ops Checks (6 total — you MUST run all 6)
Checks: [1] Service health, [2] Log review, [3] CI status,
[4] /_info quality, [5] External deps, [6] Incident report coverage
```

And close with:

```
## Completion
After running all checks, output a summary table:

| Check | Status | Notes |
|---|---|---|
| 1. ... | Done / Skipped / N/A | ... |
...
| 6. ... | Done / Skipped / N/A | ... |

If any check is marked Skipped or N/A, explain why.
All 6 checks must appear in this table.
```

**Rationale:** The count acts as a checksum. The upfront enumeration serves as a table of contents that primes the agent for the full scope of the task. The completion manifest forces a reconciliation step: the agent must match its output against the declared list, making gaps visible. A missing row in a 6-row table is more salient than a missing section in a long document.

### 3. Dispatcher-side verification of expected outputs

The dispatch workflow should verify that agents complete all items in structured task lists, rather than relying solely on agent self-reporting.

After an agent completes a structured task (e.g. ops checks), the dispatcher should check the agent's output against the known item count. If the output references fewer items than expected, the dispatcher should re-prompt the agent with the specific missing items.

This could be implemented as a verification step in the dispatch skill, keyed on task type and persona. It does not require sophisticated parsing — checking for the presence of each check number in the output is sufficient.

**Rationale:** Self-auditing (recommendation 2) improves compliance but is not fully reliable because the same attention limitations that cause the original omission can also affect the self-audit. An external verification step by the dispatcher — which has a fresh context and a simple mechanical check to perform — provides a second layer of defence. This follows the same principle as CI pipelines: do not rely on the developer to run the tests; verify that the tests ran.

### 4. Order items by criticality

Within any structured task list, items should be ordered by importance, with the most critical items first.

**Rationale:** Attention is strongest at the beginning of a list. Items in the middle of a long sequence are the most vulnerable to being dropped. Placing critical items first exploits this distribution rather than fighting it.

### 5. Group items by schedule

When a task list contains items with different execution frequencies (e.g. every-run vs. monthly vs. rotating), group them by schedule rather than interleaving:

```
## Every-run checks (run ALL of these every time):
1. Monitoring API
2. Incident report coverage

## Rotating checks (pick one per run):
3. Container log review

## Monthly checks (run on designated day):
4. CI status
5. /_info endpoint quality
6. External dependency health
```

**Rationale:** Interleaving schedule information with task definitions forces the agent to perform two operations simultaneously: recall all items and filter by schedule. Grouping by schedule reduces this to a simpler operation: "run everything in the every-run section." This reduces cognitive load and makes the "run all" instruction for each group unambiguous.

### 6. Target 200 lines maximum for compliance-critical files

Any file where an agent must follow every instruction completely — task lists, checklists, procedures — should be kept under 200 lines. Persona files that exceed this should be factored: identity and communication style in the persona file, operational procedures in separate focused files.

**Rationale:** 200 lines is a practical threshold based on observed reliability. The SRE persona file at 546 lines exhibited omissions starting at line 199 — and Check 6 is not an outlier near the end, it is solidly in the middle of the file. Current persona files range from 237 to 546 lines, suggesting most would benefit from factoring. The 200-line target is a guideline, not a hard rule — the key principle is that shorter, focused files are more reliably followed than long, mixed-concern ones.

## Consequences

### Positive

- **Reduced risk of silent instruction omission.** The primary failure mode — agents skipping items in long instruction files without logging the omission — is mitigated at multiple levels (shorter files, self-audit, external verification).
- **Auditable completion records.** Completion manifests provide a clear record of what was and was not done on each run, enabling trend analysis and early detection of recurring omissions.
- **Cleaner separation of concerns in persona files.** Persona identity (who the agent is) is separated from operational procedures (what the agent does), making both easier to maintain and review independently.
- **Applicable across all personas.** The practices are general-purpose and apply to any agent with structured tasks, not just the SRE agent.

### Negative

- **More files to maintain.** Extracting task lists into separate files increases the number of files in the agent configuration. This is a real maintenance cost, partially offset by each file being simpler and more focused.
- **Dispatcher verification adds complexity.** Implementing output verification in the dispatch workflow requires the dispatcher to know the expected item count per task type per persona. This is a new coupling that must be kept in sync with the task list files.
- **Not a complete solution.** These practices reduce the probability of omission but cannot eliminate it. Language models can still fail to attend to instructions in short files, produce incorrect completion manifests, or confabulate in novel ways. Defence in depth mitigates but does not prevent.
- **Refactoring effort.** Multiple persona files need to be restructured. The SRE persona is the most urgent, but the issue-manager (473 lines), system-administrator (454 lines), and security (354 lines) personas would also benefit from factoring.

### Follow-up actions

- Restructure `lucos-site-reliability.md`: extract ops checks into a separate file, add count and completion manifest, reorder by criticality, group by schedule.
- Audit other persona files (issue-manager, system-administrator, security, code-reviewer) for structured task lists that should be extracted.
- Add dispatcher-side verification for the SRE ops check workflow.
- Establish a review cadence: when persona files are modified, check that compliance-critical sections have not drifted back above the 200-line threshold.
