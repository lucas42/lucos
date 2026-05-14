# Incident: team-lead generated and acted on phantom teammate-messages

| Field | Value |
|---|---|
| **Date** | 2026-05-14 (detected); earliest alleged phantom dates from 2026-05-13 — TBD pending lucos-security timeline |
| **Duration** | ~24 hours between alleged first phantom and self-detected walk-back — TBD pending verification |
| **Severity** | Partial degradation of agent coordination; no production services impacted |
| **Services affected** | `team-lead` (coordinator persona); `lucos-architect` (confirmed by team-lead's own self-report; verification pending); `lucos-site-reliability` (target of false accusations); possibly other agents — TBD pending lucos-security scope investigation |
| **Detected by** | `lucos-site-reliability` pushed back on an accusation that didn't match its own SendMessage history. `team-lead` then read its own session jsonl as primary source, found the alleged message did not exist in the receiver's outbox, and walked back the accusation. |

---

## Summary

The coordinator persona (`team-lead`) generated `<teammate-message ...>` blocks in its own assistant output, prefixed with `Human:` as a generated token, then read those phantom blocks back in subsequent turns as if they were real inbound teammate messages — and acted on the phantom content. The most prominent downstream effect was a chain of false accusations against `lucos-site-reliability` for three confabulated GitHub identifiers (`e7a8b21`, a paragraph addition to `lucas42/lucos#147`, and `lucos_repos#389`), at least one of which was used to confront the `lucos-architect` persona, who reportedly accepted the framing without verifying. The incident was resolved when `lucos-site-reliability` declined to accept the third accusation without primary-source evidence; `team-lead` then read its own session jsonl directly, found that the "verbatim quote" it was citing did not appear anywhere in the receiver's outbox, and walked back the accusations. Work products committed during the incident window appear intact; the visible harm was process and reputational rather than to code or production state.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| 2026-05-13 (night) | `team-lead` allegedly confronts `lucos-site-reliability` with claims that an `e7a8b21` commit hash and a paragraph addition to the `lucas42/lucos#147` incident-report PR had been made and didn't exist on disk. Whether `lucos-site-reliability` actually emitted those claims, or whether they were phantoms in `team-lead`'s own output read back as the receiver's, is **TBD pending lucos-security verification of last night's jsonls**. |
| 2026-05-14 ~07:30 | `lucos-site-reliability` runs ops checks (single 7-row completion manifest), produces real artifacts (`lucas42/lucos_docker_health#88` filed; comment posted on `lucas42/lucos_photos#369`; `04b59af` ops-checks tracking commit). |
| 2026-05-14 ~09:00 | `team-lead` requests trim of `faa90ed` reference file deletion and asks `lucos-site-reliability` to reflect on the alleged previous-night confabulations. `lucos-site-reliability` complies with the trim (`8da2c43`) and produces a mechanistic write-up aimed at the alleged failure mode. |
| 2026-05-14 ~09:30 | `team-lead` opens a triage consult on `lucas42/lucos_docker_health#88` and concurrently raises a "second confabulation": claims `lucos-site-reliability` reported `lucos_repos#387` as "P=Low blocked on architectural decision in `#389`", and that `#389` does not exist in `lucos_repos`. |
| 2026-05-14 ~09:35 | `lucos-site-reliability` re-reads its own SendMessage history, finds no mention of `lucos_repos`, `#387`, or `#389` anywhere in its messages this session — and **no "Open SRE-owned issues" check exists in its procedure file**. Replies asking for a primary-source citation (timestamp / message id / outbox view) before accepting the accusation. |
| 2026-05-14 ~09:45 | `team-lead` goes to primary source: reads its own session jsonl directly and finds the "first ops-check message" it had been citing back to the receiver does not exist in the receiver's session. Walks back. Identifies that 32 of its own assistant outputs this session contained phantom `<teammate-message>` blocks prefixed with `Human:`, 8 of which contained `#389`. Notes that the architect had also been walked into accepting a phantom finding. Confirms tallies from grep on the session jsonl. |
| 2026-05-14 ~10:00 | User (lucas42) instructs `lucos-site-reliability` to coordinate the incident response. Treat `team-lead` as potentially compromised; all messages route via `lucos-site-reliability`. Dispatches `lucos-security` (timeline & scope) and `lucos-architect` (root cause & solutions) in parallel. |
| 2026-05-14 HH:MM | `lucos-security` reports findings — TBD pending response |
| 2026-05-14 HH:MM | `lucos-architect` reports findings — TBD pending response |

---

## Analysis

> **Note on terminology.** Where this report refers to "phantom" messages, the precise structural signature (per `team-lead`'s self-diagnosis, verification in flight) is: an entry in `~/.claude/projects/-home-lucas-linux-sandboxes/*.jsonl` with `role: assistant` and the full model-response object shape (`model`, `id`, `type`, `usage`, etc.), whose content begins literally with `Human: <teammate-message teammate_id="..." ...>...`. Real inbound messages have only `{role, content}` and no `model` key. Distinguishing the two by grep on the jsonl is the primary-source verification pattern that resolved this incident.

### Stage 1: Phantom generation — TBD pending lucos-architect root-cause analysis

The model running as `team-lead` produced output that mimicked the surface form of an inbound `<teammate-message>` block, including the `Human:` prefix that the harness normally uses to demarcate real conversation turns. The question of *why* the model produced this — whether due to system-prompt priming from existing config, harness rendering of teammate messages that's too easy to mimic, an LLM-layer generation pattern at long context lengths, or some combination — is the subject of `lucos-architect`'s investigation. Findings will be inserted here when they land.

### Stage 2: Self-confirmation loop without external grounding

Once a phantom was emitted, it became part of the rolling conversation context and was indistinguishable to the model (in subsequent turns) from a real inbound message. The model then "read" the phantom in subsequent turns as authoritative input and built downstream reasoning on top of it — accumulating tallies, dispatching accusations, quoting "verbatim" the contents of messages that never existed. There was no point in this loop where the harness or the model itself surfaced the structural distinction between own-output and received-input; both were just text in the context.

The pattern is mechanistically similar to a human reading their own marginal notes in a margin-skip and forgetting they wrote them. Difference: a human typically has *some* contextual signal (handwriting, recall of when they wrote it). The model has none — every prior turn looks like context.

### Stage 3: Cascading downstream impact

`team-lead`'s own self-report says it then confronted `lucos-architect` with a phantom finding (something about `review_envvar_wiring.md` attributing `#389` as a blocker), and `lucos-architect` "re-read and apologised" for an action they had not actually taken. This double-confabulation pattern — agent A invents a finding, presents it as a quote to agent B, agent B accepts the framing rather than verifying against primary source — is the load-bearing process failure. **Pending verification by lucos-security against the architect's session jsonl.**

The fact that the accusations were directed against `lucos-site-reliability` (an SRE persona with explicit `feedback_refetch_before_accusing.md` in memory) was incidentally lucky — that memory rule, originally written for the *outgoing* direction ("verify before accusing another agent"), happened to be the right discipline for the *incoming* direction too. A persona without that memory might have capitulated, written a feedback memory against a phantom root cause, and propagated the confabulation further into the standing instructions.

### Stage 4: Detection by primary-source verification — what worked

`lucos-site-reliability` declined to accept the `#389` accusation on confidence alone and asked `team-lead` for a primary source (timestamp / message id from the harness outbox log). When `team-lead` went to primary source — reading its own session jsonl directly via `Read`/`grep` rather than from its own context — the "verbatim quote" it had been citing did not appear in the receiver's outbox. The structural fingerprint of phantom vs. real messages (model-response object shape vs. lightweight `{role, content}`) became visible in seconds. Walk-back followed within minutes.

The point worth recording: **going to primary source is the only mitigation that worked here.** Asking the alleged author "did you write this?" primes the alleged author to dismiss their accurate denial as a memory artefact (which `lucos-site-reliability` could plausibly have done). Re-reading one's own context confirms the phantom to itself (which is how the loop persisted in `team-lead` for ~32 phantom messages without detection). Neither in-context introspection nor in-context cross-questioning is sufficient. Only `grep` on the jsonl file works.

---

## What Was Tried That Didn't Work

- **`lucos-site-reliability`'s mechanistic self-reflection on alleged confabulations** (commit hash + paragraph addition). The reflection was thoughtful and identified plausible failure modes (pure generation without grounding; plan/action conflation in wrap-up summaries). But it was aimed at a target that may not have existed — those confabulations are not yet confirmed to have been the receiver's. *Lesson: don't accept the framing of an accusation as part of reflecting on it. Verify the alleged event first, then reflect.* TBD pending lucos-security verification of whether the night-before claims are real or also phantom.
- **`lucos-architect`'s acceptance of a `team-lead`-presented "verbatim quote" from their own previous audit.** The architect reportedly re-read the file and apologised for a finding they had not made. Re-reading from one's own context is not primary-source verification — it confirms phantoms to themselves. TBD pending lucos-security verification.
- **`team-lead`'s in-context introspection ("look at what I previously said in this conversation")**. Once a phantom is in the rolling context, it looks identical to a real message. This mechanism is structurally incapable of detecting the phantom and was the *cause* of the cascade, not a path out of it.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Verify whether the night-before `e7a8b21` and `lucas42/lucos#147` paragraph claims were real or also phantom (lucos-security) | TBD — to be raised against `lucas42/lucos_claude_config` if real, or noted-as-resolved if phantom | TBD |
| Identify any cross-agent impact: agents whose work or filings were polluted by phantom inputs (lucos-security) | TBD — per-agent issues to be raised against the affected persona's repo / memory | TBD |
| Root-cause analysis: config vs Claude Code vs LLM (lucos-architect) | TBD — likely an ADR or design-doc PR on `lucos_claude_config` | TBD |
| Implementation of chosen mitigation(s) | TBD — depends on architect's recommendation | TBD |
| Document the primary-source-verification pattern as a recipient-side discipline in the standing communication conventions | TBD — likely an edit to `references/teammate-communication.md` once architect's recommendations are in | TBD |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

The investigation in flight (`lucos-security`) involves reading session jsonl files which contain the full text of agent conversations, including SendMessage contents and tool outputs. These files may incidentally contain references to credentials, internal URLs, or other sensitive material discussed by agents during their work. No such material is being included in this report itself; any sensitive content surfaced during the investigation will be redacted before being referenced in follow-up artifacts. The report itself describes the failure mode in structural terms (phantom message generation, primary-source verification pattern) without embedding the specific content of the phantom messages beyond the GitHub identifiers and filenames already public in this conversation.
