# ADR-0012: External delegation topology for off-site DNS resilience

**Date:** 2026-06-08
**Status:** Proposed
**Discussion:** [lucas42/lucos_dns#107](https://github.com/lucas42/lucos_dns/issues/107)
**Incident:** [2026-06-07 estate-wide DNS outage](../incidents/2026-06-07-estate-wide-dns-outage.md)
**Extends:** [ADR-0003](0003-secondary-dns-nameserver.md) (secondary nameserver), [ADR-0010](0010-dns-primary-secondary-as-separate-systems.md) (primary/secondary as separate systems)

## Context

ADR-0003 decided that lucos should run a secondary authoritative nameserver on `xwing`; ADR-0010 modelled the primary and secondary as two separate systems/repos. Both addressed *serving* the zones from a second host. **Neither addressed how external resolvers discover and reach that second host** — the parent-delegation topology.

The [2026-06-07 estate-wide DNS outage](../incidents/2026-06-07-estate-wide-dns-outage.md) exposed the gap. A single fault on `avalon` — a stale generated `l42.eu` apex zone, rejected by BIND on a deploy restart — took out *all* authoritative DNS for the estate, because every externally-visible nameserver for `l42.eu` resolves to `avalon`.

The delegation, as verified by lucos-site-reliability on 2026-06-08 via the `.eu` TLD servers (facts below are their verification, not independently re-checked here):

```
l42.eu.  NS  dns.l42.eu.            → 178.32.218.44 / 2001:41d0:8:dc2c::1   (avalon)
l42.eu.  NS  ns1.lukeblaney.co.uk.  → CNAME → dns.l42.eu. → (avalon)
```

So both delegation nameservers resolve to the single `avalon` host. The new `xwing` secondary, `dns2.l42.eu`, exists only as an **in-zone apex `NS` record** — which a resolver priming `l42.eu` from the `.eu` parent never sees.

Two distinct faults:

1. **No second host in the external NS set.** Even once `xwing` transfers zones successfully (lucas42/lucos_dns#103), external resolvers still only ever reach `avalon`, because the parent delegation contains `avalon` twice.
2. **The "off-site" NS is circularly dependent and non-conformant.** `ns1.lukeblaney.co.uk` is (a) a **CNAME**, which is forbidden as an NS target (RFC 2181 §10.3 — an NS name must have address records, not be an alias), and (b) **out-of-bailiwick** under `lukeblaney.co.uk` — itself a lucos-served zone — **with no glue**, so it can only be resolved by querying the same `avalon` servers that are down. The backup name is unresolvable exactly when it is needed.

## Decision

Set the `.eu` parent delegation for `l42.eu` to **two distinct hosts, each reachable independently of `l42.eu`'s own servers via registry-served glue**:

- Delegation NS set = **{ `dns.l42.eu`, `dns2.l42.eu` }** — `avalon` and `xwing`, not `avalon` twice.
- Glue at the `.eu` registry: `dns.l42.eu` → `avalon` A/AAAA; `dns2.l42.eu` → `xwing` A/AAAA.
- **Remove `ns1.lukeblaney.co.uk` from the delegation** and retire the CNAME-as-NS arrangement.

### Rationale: glue, not an independent name, provides outage-survivable resolvability

The key insight is that the circular dependency disabling `ns1.lukeblaney.co.uk` is a property of *being an unglued, out-of-bailiwick CNAME*, **not** of being named under `l42.eu`.

`dns2.l42.eu` is **in-bailiwick** (under `l42.eu`), so the `.eu` registry **must** carry glue (A/AAAA) for it. That glue is served by the `.eu` registry — wholly independent of `avalon` and `xwing`. A resolver priming `l42.eu` obtains both nameservers' addresses directly from `.eu` and **never needs to resolve `dns2.l42.eu` by name**. The circular dependency therefore does not arise for an in-bailiwick glued name.

**Failure-mode check against 2026-06-07:** with both names glued at `.eu`, a resolver priming `l42.eu` receives both glue addresses from `.eu`, queries `avalon` (SERVFAILing on the broken apex) *and* `xwing` (serving last-known-good), and gets a valid answer from `xwing`. Resolution never depends on `l42.eu`'s own servers being up — which is the external resilience ADR-0003 intended.

### Consistency with the in-zone apex

The in-zone apex `NS` RRset (`dns.l42.eu` + `dns2.l42.eu`, the latter added by the primary-side work in lucas42/lucos_dns#79) must match this parent-delegation set. The three things — parent NS set, parent glue, and in-zone apex NS — are then mutually consistent.

## Sequencing

The registrar change (lucas42/lucos#111) must land **only after**:

1. the secondary is standing (lucas42/lucos_dns_secondary#1),
2. TSIG-authenticated transfer works (lucas42/lucos_dns#103), and
3. `xwing` is verified serving correct answers for all five zones.

Adding `dns2.l42.eu` to the delegation before that would hand external resolvers a glue record for a server that SERVFAILs, degrading rather than improving availability. lucas42/lucos#111 is already blocked on this ADR / lucas42/lucos_dns#107.

The removal of `ns1.lukeblaney.co.uk` carries a pre-flight check (see Follow-up actions) that belongs in lucas42/lucos#111's scope: confirm nothing else references that name before deleting it.

## Alternatives considered

1. **Out-of-bailiwick secondary name** in a third-party / registrar-hosted zone (the literal "give it a name that doesn't resolve through lucos zones" suggestion). Works, but adds a dependency on an external DNS provider for the secondary's name to buy resolvability that registry glue already provides for free for an in-bailiwick name. More moving parts, no extra resilience. **Rejected.**
2. **Keep `ns1.lukeblaney.co.uk`, repoint it at `xwing`.** Still a CNAME-as-NS (RFC-non-conformant) and still out-of-bailiwick under a lucos-served zone, so still unresolvable during an `l42.eu` outage unless given its own glue — at which point `dns2.l42.eu` is the simpler in-bailiwick choice. **Rejected.**
3. **In-zone apex NS only (status quo plus lucas42/lucos_dns#79).** Insufficient: apex `NS` records are invisible to resolvers priming from the parent. Only the `.eu` delegation NS set plus glue determines who the outside world reaches. **Rejected** as the sole measure (it remains necessary for consistency).

## Consequences

### Positive

- **Genuine off-site resilience.** An `avalon` outage no longer removes all authoritative DNS, once the secondary is in the delegation. This is the payoff ADR-0003 was reaching for.
- **Standards-conformant and non-circular.** Resolves the RFC 2181 §10.3 NS→CNAME non-conformance and the circular-name dependency in one move.
- **No third-party DNS dependency.** Uses standard registry glue rather than an externally-hosted name.
- **Internally consistent.** External NS set, registry glue, and in-zone apex NS all agree.

### Negative

- **Glue must be kept correct at the registrar.** A stale `dns2.l42.eu` glue record (e.g. after an `xwing` IP change) would misdirect a share of external queries. Registrar glue changes are manual and rare; accept and document, and treat an `xwing` address change as a registrar-touch event.
- **The registrar change is outside CI/automation.** It is a one-off manual change at the registrar, gated behind the verification sequencing above.
- **Two delegated hosts split external query load.** `xwing` must be provisioned to answer real external query volume, not merely transfer zones. As a full authoritative secondary this is expected, but it is a load it does not bear today.

### Operational correlation (unchanged from ADR-0003/0010)

`avalon` and `xwing` still share a human operator and image lineage. This ADR removes the single-host *delegation* SPOF; it does not eliminate correlated-failure risk. Standard BIND last-known-good behaviour on a bad transfer (lucas42/lucos_dns#104) remains the backstop.

## Follow-up actions

(The coordinator owns labels, priority, and board placement.)

- **lucas42/lucos#111 (registrar NS + glue)** — execute this delegation (`{ dns.l42.eu, dns2.l42.eu }` + glue, drop `ns1.lukeblaney.co.uk`) once the sequencing preconditions are met. Already tracked and blocked on this ADR; no new issue needed. Its scope should explicitly include the `ns1.lukeblaney.co.uk` removal pre-flight (confirm no other referents — other zones' NS sets, monitoring — before deleting).
