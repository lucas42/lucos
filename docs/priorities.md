# Strategic Priorities

High-level guidance for prioritising work across lucos repositories. Agents should consult this file when setting the Priority field during triage.

**Last updated**: 2026-06-01

---

## Active priorities (in order)

### 1. lucos_firewall

Design and rollout of the `lucos_firewall` service, per ADR-0007 — the current top priority. The repo is bootstrapped and the `public_ports` schema has landed in `lucos_configy`. Remaining work: populating `public_ports` for the public-facing services, implementing the firewall container (Go) that generates and applies iptables/ip6tables rules from configy, RNDC tightening in `lucos_dns`, and the progressive deployment to avalon → xwing → salvare. Issues that progress the firewall implementation or its rollout should be Priority = High.

---

## Paused repositories

The following repositories have all non-critical work paused. Issues in these repos should generally be set to Priority = Low, or left unprioritised. Critical security updates are the only exception.

### lucos_auth

All work paused except critical security updates. The service is planned to be replaced entirely with something based around passkeys. New feature work would be wasted effort.

---

## How to use this file

The Priority field is set on **every triaged issue** — including refinement-phase issues (Status = Ideation or Awaiting Decision), not just Status = Ready. This ensures refinement work is also prioritised.

When setting Priority during triage:

- **Critical**: A production service is completely down and users are affected right now. This is for full service outages only — not for "very important" features, not for degraded performance, and not for bugs that have workarounds. If you are unsure whether something qualifies, it is not critical. See "Preventing priority inflation" below.
- **High**: Issues that fall under any of the active priorities listed above, issues causing a current alert (see "Active alerts" below), or are urgent regardless of area.
- **Medium**: Important issues not in the top priority areas.
- **Low**: Everything else, including issues in paused repositories.

If an issue in a paused repository has a genuine critical security concern, it may still warrant Priority = High — use judgement.

### Ordering within a priority level

Within a given Priority field value (Critical / High / Medium / Low), order issues by:

1. **Strategic tier they fall under.** Issues advancing active priority #1 sit above issues advancing active priority #2, and so on. Issues that do not fall under any active strategic priority sit below issues that do.
2. **Paused-repo issues sort last** within their Priority level, below all other no-tier issues. Once a repository is paused, its issues should only be picked up if the queue is otherwise clear at their Priority level.
3. **Age, oldest first**, within the same strategic tier (or within the paused-repo band).

Strategic-tier ordering only applies *within* a Priority field value — it does not promote a Medium issue above a High one. To promote across Priority levels, change the Priority field. Two issues can therefore be in the same tier but at different Priority levels; the Priority field still wins.

Ordering can be overridden by lucas42 via manual board repositioning, which is authoritative — do not undo a manual reposition just because the strategic-tier sort would have placed it differently.

### Preventing priority inflation

Priority = Critical exists to ensure live outages are always picked up ahead of everything else, no matter how old other issues are. To keep it effective:

- **Do not use it for features**, no matter how important. Use Priority = High instead.
- **Do not use it for bugs that have workarounds.** If users can still accomplish their goal, it is not critical.
- **Do not use it for degraded performance.** Slow is not down.
- **Remove it as soon as the outage is resolved.** Critical issues should not linger — they are either actively being fixed or already closed.

If Priority = Critical is used too broadly, it loses its signal and the queue becomes indistinguishable from Priority = High.

### Active alerts

Any issue that is currently causing an alert — a monitoring alert, a failing health check, a red CI status blocking deploys, or similar — should be triaged at least to Priority = High, regardless of which repository it is in. An active alert overrides the normal priority framework because it represents ongoing impact.

If the alert constitutes a full production outage (service down, users affected), use Priority = Critical as per the existing rules above.

This applies both at initial triage and when an existing issue escalates in impact. If an issue was originally set at Priority = Low or Medium but is now causing an active alert, it should be raised to at least Priority = High.

### Re-assessment after lucas42 input

When lucas42 gives input on an issue (e.g. a comment, a decision, or a reaction), re-assess the Priority. The scope or urgency may have changed, or lucas42 may have explicitly stated a priority.

### Priority override rules

- **lucas42's priority calls override this file.** lucas42 is the repo owner and has final say. If lucas42 explicitly states a priority level for an issue, apply that priority regardless of what this document says.
- **Priority calls from others** (including agents) should be considered within the context of the strategic priorities defined here. They do not automatically override this framework — they are input, not directives.
