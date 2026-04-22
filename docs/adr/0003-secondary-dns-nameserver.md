# ADR-0003: Secondary authoritative DNS nameserver off-host

**Date:** 2026-04-22
**Status:** Accepted
**Discussion:** https://github.com/lucas42/lucos/issues/109

## Context

The authoritative nameservers for `l42.eu` (and the other lucos-managed zones: `s.l42.eu`, `lukeblaney.co.uk`, `rowanblaney.co.uk`, `tfluke.uk`) have historically run on a single host: `avalon`. On 2026-04-22, a Docker daemon restart on avalon left ~40 containers in an orphaned containerd-task state — including `lucos_dns_bind`. Every `*.l42.eu` hostname went SERVFAIL at public resolvers for the duration of the incident, amplifying a fleet-wide container-state problem into a full external outage (see `docs/incidents/2026-04-22-avalon-containerd-task-orphan.md`).

During that incident, investigation of the zone files revealed a further issue: each lucos-managed zone has only **one** `NS` record in its apex RRset:

- `l42.eu` zone: `@ IN NS dns.l42.eu.`
- `s.l42.eu` zone: `@ IN NS dns.l42.eu.`
- `lukeblaney.co.uk` zone: `@ IN NS dns.l42.eu.` and a separate `ns1 IN CNAME dns.l42.eu.` (not a second nameserver)

`dns.l42.eu` is the only authoritative nameserver. The apparent secondary `ns1.lukeblaney.co.uk` is a CNAME back to `dns.l42.eu`, and in any case does not appear in any zone's NS RRset. RFC 2182 (BCP 16) — *"Selection and Operation of Secondary DNS Servers"* — has recommended topologically diverse secondaries since 1997. lucos has never had any NS-level redundancy at all.

The core question: *should the authoritative nameservers for a zone be co-hosted with services in that zone?*

Four options were considered in lucas42/lucos#109:

1. **Keep the current setup.** Accept the amplifier effect. Ruled out: the amplifier is a design flaw violating a foundational DNS principle, not an acceptable trade-off.
2. **Add an off-host secondary NS.** Slave to the current BIND primary via AXFR/IXFR.
3. **Move DNS entirely to a managed provider.** Gain anycast and vendor SLA, lose the `lucos_dns_sync` → BIND config-as-code loop.
4. **Split-brain: separate infra-DNS from service-DNS.** Ruled out: more operational complexity, no meaningful benefit over option 2.

## Decision

We will add one secondary authoritative nameserver, hosted on **`xwing`**, slaving from the `avalon` BIND primary via TSIG-authenticated AXFR/IXFR. The secondary will be exposed publicly as `dns2.l42.eu` and listed as a second `NS` record in every lucos-managed zone, with appropriate glue at the registrar.

`xwing` is suitable because it is on a genuinely independent network from `avalon`:

- `avalon`: `178.32.218.44` — OVH datacentre, France
- `xwing`: `152.37.104.10` / `2a01:4b00:8598:5a00:ba27:ebff:fe83:e1ee` — Zen Internet, UK premises

When `avalon`'s network, datacentre, or containerd state fails, `xwing`'s connectivity is not correlated with the cause. That is the property RFC 2182 calls for and the property the 2026-04-22 incident exposed as missing.

We reject the alternative options for the following reasons:

- **Status quo (option 1)** violates RFC 2182 with no offsetting benefit.
- **Managed DNS provider (option 3)** would replace `lucos_dns_sync` with per-provider API integration, introduce a third-party vendor dependency for every zone change, and cost money. The scale of lucos does not yet justify anycast.
- **Split-brain (option 4)** adds operational complexity for marginal benefit — every zone matters once a user is trying to reach *any* lucos service.
- **Hosted secondary-NS service** (a sub-option of option 2) was considered but rejected in favour of `xwing` on the grounds that `xwing` is already in the lucos CI/deploy loop and introduces no new operational surface or cost.

### Implementation sketch

The detailed work lives in follow-up issues (linked in the PR). In outline:

- **Secondary BIND on xwing.** A second `lucos_dns` deployment configured via env var to run BIND in `type secondary` mode for the lucos-managed zones. Same container image as the primary; only configuration differs. `lucos_dns_sync` remains a primary-only component and is not deployed to xwing.
- **Zone transfer.** TSIG-authenticated AXFR/IXFR from `avalon` primary. `allow-transfer` and `also-notify` on the primary are narrowed to the secondary's IP(s) and TSIG key.
- **Zone file changes.** `l42.eu.jinja` and `s.l42.eu.jinja` (and the static `lukeblaney.co.uk`, `rowanblaney.co.uk`, `tfluke.uk` zone files) gain a second `NS` record for `dns2.l42.eu` and a glue `A` (and `AAAA`) record pointing to xwing.
- **Registrar configuration.** NS records and glue at the registrar are updated for each managed domain. This is a manual step and is lucas42's to execute.
- **Monitoring.** A new monitoring check compares SOA serial between primary and secondary to catch silent zone-transfer failures. Without this, we would discover a broken secondary only during the next primary outage — which is exactly the failure mode we are trying to prevent.

### Related follow-up

Separately from the secondary NS work, the `l42.eu` zone's `$TTL` is `300` (5 minutes). This sharply shortened the cache window during the 2026-04-22 outage — public resolvers stopped serving cached answers ~5 minutes in. For records that do not churn (MX, apex A, static CNAMEs to external providers), a longer TTL would have reduced external impact. This is a separate, complementary change tracked as its own follow-up issue.

## Consequences

### Positive

- **Single-point-of-failure removed from the DNS layer.** When `avalon` is down, cached `l42.eu` answers continue to be served by the secondary for at least the zone SOA `expire` window (currently 28 days).
- **Complies with RFC 2182.** Nameservers are now topologically diverse — different provider, different country, different operational environment.
- **No new hosting cost.** `xwing` already exists and is already part of the lucos estate.
- **No new vendor dependency.** The config-as-code loop through `lucos_dns_sync` is preserved; the secondary is a slave within the same BIND/lucos codebase.
- **Debuggable failure modes.** Zone-transfer failures produce standard BIND log lines and are directly monitorable via SOA serial comparison.

### Negative

- **Operational correlation through shared code and ops.** The primary and secondary share the same `lucos_dns` codebase, the same CI pipeline, and the same human operator. A bug in the image that broke BIND startup would break both. This is mitigated partly by the fact that standard BIND secondary behaviour on receipt of an invalid zone transfer is to reject and continue serving last-known-good data — but we should verify that behaviour during implementation.
- **Consumer-ISP uptime for the secondary.** `xwing`'s uplink reliability is probably lower than a datacentre's. For a *secondary* this is acceptable: the requirement is that the secondary is up when the primary is down, not that the secondary is up all the time. The networks are independent, which is what matters.
- **Registrar configuration now lives outside config-as-code.** NS records and glue at the registrar must be updated manually. There is no practical automated path because the registrar is outside our infrastructure. This is a one-time cost at setup (plus occasional re-verification) rather than an ongoing friction point.
- **Not anycast.** Resolvers will try one NS and then the other if the first fails. For the volume of DNS queries lucos sees, this is fine.

### Follow-up actions

Implementation work is tracked in separate issues linked from the PR. Summary:

- `lucos_dns` — Deploy BIND secondary to xwing, add TSIG, update zone templates (`priority:low`)
- `lucos` — Update registrar NS records and glue (`priority:low`, manual step for lucas42)
- `lucos_monitoring` — SOA serial consistency check between primary and secondary (`priority:low`)
- `lucos_dns` — Raise `$TTL` for static records in the `l42.eu` zone (`priority:low`)
