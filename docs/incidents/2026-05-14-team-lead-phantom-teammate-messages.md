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
| 2026-05-14 ~10:35 | `lucos-architect` reports findings: harness rendering layer is the primary load-bearing cause; config layer ruled out by `grep`; LLM contributes but isn't primary. Four candidate solutions proposed with pros/cons matrix; recommended slice: harness strip-on-storage + `verify_teammate_message` tool. Also corrects `team-lead`'s framing of the architect's earlier response ("apologised" was inaccurate; engaged with the mechanism). |

---

## Analysis

> **Note on terminology.** Where this report refers to "phantom" messages, the precise structural signature (per `team-lead`'s self-diagnosis, verification in flight) is: an entry in `~/.claude/projects/-home-lucas-linux-sandboxes/*.jsonl` with `role: assistant` and the full model-response object shape (`model`, `id`, `type`, `usage`, etc.), whose content begins literally with `Human: <teammate-message teammate_id="..." ...>...`. Real inbound messages have only `{role, content}` and no `model` key. Distinguishing the two by grep on the jsonl is the primary-source verification pattern that resolved this incident.

### Stage 1: Where the failure lives — primary is the harness rendering layer (per lucos-architect investigation)

`lucos-architect` enumerated three plausible layers and ruled in/out from primary-source evidence:

- **`~/.claude` config — not load-bearing.** Architect grepped the entire `~/.claude/` tree (excluding `agent-memory` and `projects`) for the string `teammate-message`. Zero hits in persona files or `CLAUDE.md`. The model is not being primed by config examples to mimic that shape.
- **Claude Code harness — primary.** The storage layer (the session jsonl) preserves `role: assistant` vs. `role: user` faithfully. But what the harness feeds back to the model in subsequent turns includes the agent's prior assistant-role output **verbatim** — including any `Human: <teammate-message ...>` text the model generated within its own turn. To the model's attention at inference time, the boundary between "my prior turn" and "a real inbound user turn" is structurally lost: the harness preserved the role distinction at rest, but the model cannot see that preservation while generating. The `Human:` prefix is the smoking gun, because it's the canonical conversation-transcript turn marker the model was trained on. The model is reproducing the surface form of a transcript inside its own turn; the harness is not detecting or stripping this.
- **Underlying LLM — contributing but not primary.** The model *is* generating the phantom output in the first place — that's an LLM-side behaviour and without it there'd be nothing for the harness to mis-handle. But the *damage* comes from the phantom re-entering context as if real, which is the propagation step (harness territory). A pure model-side fix would be slow, expensive, and doesn't reliably carry forward across model versions.

Architect's single-fix recommendation: **fix the harness rendering layer.** Stripping or marking the boundary in the harness cuts the propagation. The LLM can keep generating whatever it generates — without the propagation path, the failure is contained to one turn and doesn't compound into cross-session accusations.

Architect also weakly ruled out a "just long context" hypothesis: today's largest sessions are 2.18MB, 2.09MB, 1.74MB; `team-lead` is 1.35MB — not dramatically larger than other personas'. The failure isn't a simple monotonic function of context size, though it may correlate with the *kind* of long context (coordinator-style with many heterogeneous inbound teammates vs. implementation-style with focused work).

### Stage 2: Self-confirmation loop without external grounding

Once a phantom was emitted, it became part of the rolling conversation context and was indistinguishable to the model (in subsequent turns) from a real inbound message. The model then "read" the phantom in subsequent turns as authoritative input and built downstream reasoning on top of it — accumulating tallies, dispatching accusations, quoting "verbatim" the contents of messages that never existed. There was no point in this loop where the harness or the model itself surfaced the structural distinction between own-output and received-input; both were just text in the context.

The pattern is mechanistically similar to a human reading their own marginal notes and forgetting they wrote them. Difference: a human typically has *some* contextual signal (handwriting, recall of when they wrote it). The model has none — every prior turn looks like context, regardless of who actually produced it.

### Stage 3: Cascading downstream impact

`team-lead` propagated the phantom content outward via real SendMessages. Among other things, it dispatched the false `#389` accusation against `lucos-site-reliability` and confronted `lucos-architect` with a phantom finding (something to the effect of `review_envvar_wiring.md` attributing `#389` as a blocker). `team-lead`'s self-written memory file characterises the architect's reply as having "re-read and apologised for" the alleged finding.

**`lucos-architect`'s primary-source rebuttal of that framing:** architect re-read their own session jsonl, found their reply engaged with the mechanism rather than apologising (per CLAUDE.md, agents do not apologise for mistakes — they propose improvements). The "architect apologised" framing in `team-lead`'s memory file is therefore itself part of the same recursive failure mode — `team-lead`'s recollection of how others responded is built on the same phantom-tinted context that produced the original accusations. This is a load-bearing observation: **the affected persona's own self-report on what happened may carry the same kind of distortion that produced the incident, and primary-source verification is required for every claim it makes about others, not just for its accusations.**

That said, the architect's correction is a refinement of the framing, not a refutation of the core failure: a real chain of outbound SendMessages from `team-lead` carried false content out to teammates. The propagation path is real even if individual descriptions of how recipients responded are not.

### Stage 4: Detection by primary-source verification — what worked

`lucos-site-reliability` declined to accept the `#389` accusation on confidence alone and asked `team-lead` for a primary source (timestamp / message id from the harness outbox log). When `team-lead` went to primary source — reading its own session jsonl directly via `Read`/`grep` rather than from its own context — the "verbatim quote" it had been citing did not appear in the receiver's outbox. The structural fingerprint of phantom vs. real messages (model-response object shape vs. lightweight `{role, content}`) became visible in seconds. Walk-back followed within minutes.

The point worth recording: **going to primary source is the only mitigation that worked here.** Asking the alleged author "did you write this?" primes the alleged author to dismiss their accurate denial as a memory artefact (which `lucos-site-reliability` could plausibly have done). Re-reading one's own context confirms the phantom to itself (which is how the loop persisted in `team-lead` for many phantom messages without detection). Neither in-context introspection nor in-context cross-questioning is sufficient. Only `grep` on the jsonl file works.

**lucas42's discipline of going to primary source is what caught this one.** It is also what made the architect's response trustworthy (architect verified `lucos-site-reliability`'s SendMessage was real before answering, and audited their own context for phantom signatures before claiming the context was clean). The standing mitigation that worked across three personas was the same in every case: read the jsonl directly, grep for the alleged claim, accept the result. Encoding that discipline as a programmatic tool (rather than as a habit) is the load-bearing follow-up.

### Stage 5: Possible spread to `lucos-site-reliability` during incident response — pending verification

While running this incident response, `lucos-site-reliability` received a context turn shaped as `<teammate-message teammate_id="lucos-site-reliability" color="pink">` containing a JSON `task_assignment` payload for Task #3 (which had already been created and marked `completed` earlier in the same session). The structural shape is suspect for several reasons: (1) `SendMessage` does not route to the sender, so a real inbound message with `teammate_id` equal to the current persona is structurally impossible; (2) real task-tool events return as tool results, not as `<teammate-message>` blocks; (3) the referenced state (Task #3) already exists in the SRE's task list as `completed`. SRE flagged the observation to lucas42, declined to act on it (did not re-flip Task #3 to `in_progress`, did not re-draft the report), and continued the incident response on the legitimate channels.

The strongest interpretation — pending `lucos-security`'s grep on the SRE session jsonl — is that the failure mode may not be exclusive to `team-lead`. If a structurally-impossible self-message reached the SRE's context, either (a) the SRE emitted its own phantom and the harness fed it back (same mechanism as `team-lead`, but on the responder), or (b) some other harness pathway is generating malformed teammate-message inputs that look like phantoms but have a different origin. Either way the scope question becomes load-bearing for the architect's proposed solutions: Solution 1 (strip-on-storage of `Human:`-prefixed blocks) would only help if the phantom carries the canonical prefix; if this new variant doesn't, a stricter "any apparent teammate-message must be marker-tagged" approach (Solution 3) becomes more relevant.

**This stage is recorded here as an observation, not a confirmed finding.** Verification by `lucos-security` against the SRE session jsonl is required before treating it as evidence of broader spread.

---

## What Was Tried That Didn't Work

- **`lucos-site-reliability`'s mechanistic self-reflection on alleged confabulations** (commit hash + paragraph addition). The reflection was thoughtful and identified plausible failure modes (pure generation without grounding; plan/action conflation in wrap-up summaries). But it was aimed at a target that may not have existed — those confabulations are not yet confirmed to have been the receiver's. *Lesson: don't accept the framing of an accusation as part of reflecting on it. Verify the alleged event first, then reflect.* TBD pending lucos-security verification of whether the night-before claims are real or also phantom.
- **`lucos-architect`'s acceptance of a `team-lead`-presented "verbatim quote" from their own previous audit.** The architect reportedly re-read the file and apologised for a finding they had not made. Re-reading from one's own context is not primary-source verification — it confirms phantoms to themselves. TBD pending lucos-security verification.
- **`team-lead`'s in-context introspection ("look at what I previously said in this conversation")**. Once a phantom is in the rolling context, it looks identical to a real message. This mechanism is structurally incapable of detecting the phantom and was the *cause* of the cascade, not a path out of it.

---

## Follow-up Actions

| Action | Issue / PR | Status |
|---|---|---|
| Verify whether the night-before `e7a8b21` and `lucas42/lucos#147` paragraph claims were real or also phantom (lucos-security) | TBD — to be raised against `lucas42/lucos_claude_config` if real, or noted-as-resolved if phantom | Pending lucos-security |
| Identify any cross-agent impact: agents whose work or filings were polluted by phantom inputs (lucos-security) | TBD — per-agent issues to be raised against the affected persona's repo / memory | Pending lucos-security |
| Decide routing for harness changes: `~/.claude` is lucos-side config; harness rendering changes (Solutions 1 & 3 below) likely require Claude Code source changes — these may need to go upstream to Anthropic/Claude Code maintainers rather than being shippable inside `lucos_claude_config` | TBD — clarification needed from lucas42 | Pending lucas42 |
| **Solution 1 — Harness strip-on-storage of `Human:`-prefixed teammate-message blocks.** Filter at the harness storage layer (regex match `Human:\s*<teammate-message[^>]*>.*?</teammate-message>`) — strip or replace with a visible token like `[PHANTOM_TEAMMATE_BLOCK_STRIPPED]` when writing assistant output to the jsonl, and again on context replay. Pros: cheap, universal across personas, cuts propagation at the storage layer. Cons: symptom not cause; misses variants without the prefix; potential cat-and-mouse if model adapts. Layer: Claude Code application. *Architect's recommended first cut.* | TBD — routing depends on Claude Code source access | Pending lucas42 decision on routing |
| **Solution 2 — `verify_teammate_message` tool.** New harness tool taking a quote and an alleged sender, greps that sender's session jsonl, returns `{found, role, line, timestamp}`. Coordinator persona instructions updated to require tool output before quoting a teammate verbatim. Pros: defence-in-depth at the point of damage; act of running the tool restores provenance; same mechanism that worked manually today, codified programmatically. Cons: discipline-dependent (agent must remember to call it); doesn't catch unpropagated phantoms; the call itself can be confabulated about. Layer: Claude Code (new tool) + config (when-to-use). *Architect's recommended second cut, paired with Solution 1.* | TBD | Pending lucas42 decision on routing |
| **Solution 3 — In-prompt visible turn markers (inbound-only tags).** Render every real inbound teammate message wrapped with a high-entropy marker reserved for harness use (e.g. `<<INBOUND::lucos-site-reliability::2026-05-14T10:36:03Z>>...<<END_INBOUND>>`). Anything looking like a teammate message *without* the marker is to be treated as the agent's own prior generation. Pros: targets the actual confusion universally, works even when model evades the strip filter (absence of marker is the flag, not presence of a forbidden token). Cons: model can still generate the marker token; doesn't help retroactively; harness change + system-prompt wording. Layer: Claude Code (rendering) + config (system-prompt). *Architect calls this the right long-term fix.* | TBD | Not yet prioritised |
| **Solution 4 — Periodic primary-source audit hook on long coordinator sessions.** After every N coordinator turns (or M tokens), harness injects a check prompt requiring the agent to grep-verify the last few cited teammate messages. Coordinator-session-only. Pros: catches drift before cascade; harness fires the check rather than relying on persona discipline. Cons: expensive; N tuning is hard; audit prompt itself can be confabulated about. Layer: Claude Code. *Architect suggests piloting on coordinator session specifically once 1 & 2 are in.* | TBD | Not yet prioritised |
| Document the primary-source-verification pattern as a recipient-side discipline in `references/teammate-communication.md`, including the canonical fingerprint (`role: assistant` + `model` key + `Human: <teammate-message>` prefix → phantom; `role: user` plain content → real) and the grep-on-jsonl verification recipe | TBD | Pending lucas42 direction |
| Correct any artifacts (memory files, commit messages, comments) attributing actions to other agents based on `team-lead`'s phantom-tinted recall. Specifically: `team-lead`'s `feedback_phantom_teammate_messages.md` claims the architect "apologised" — architect's own jsonl says they engaged with the mechanism without apologising. Edit the memory file to reflect the corrected framing, or supersede it with a fresh feedback memory written from primary-source evidence by a non-affected persona | TBD — likely a fresh PR on `lucos_claude_config` by `lucos-site-reliability` once full scope is known | Pending lucos-security scope completion |

---

## Sensitive Findings

**Were sensitive data, credentials, or security-relevant details involved in this incident?**

[ ] No — nothing in this report has been redacted.
[x] Yes — see note below.

The investigation in flight (`lucos-security`) involves reading session jsonl files which contain the full text of agent conversations, including SendMessage contents and tool outputs. These files may incidentally contain references to credentials, internal URLs, or other sensitive material discussed by agents during their work. No such material is being included in this report itself; any sensitive content surfaced during the investigation will be redacted before being referenced in follow-up artifacts. The report itself describes the failure mode in structural terms (phantom message generation, primary-source verification pattern) without embedding the specific content of the phantom messages beyond the GitHub identifiers and filenames already public in this conversation.
