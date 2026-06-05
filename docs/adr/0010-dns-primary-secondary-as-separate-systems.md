# ADR-0010: Model the primary and secondary DNS nameservers as separate systems in separate repos

**Date:** 2026-06-05
**Status:** Proposed
**Discussion:** https://github.com/lucas42/lucos/issues/213
**Revises:** the implementation sketch of [ADR-0003](0003-secondary-dns-nameserver.md) (lucas42/lucos#110)

## Context

[ADR-0003](0003-secondary-dns-nameserver.md) decided *that* lucos should run a secondary authoritative nameserver on `xwing`, slaving from the `avalon` primary. Its implementation sketch assumed the secondary would be **"the same container image as the primary; only configuration differs"**, selected by an environment variable. The first implementation attempt followed that sketch:

- **lucas42/lucos_dns#95** added a `DNS_MODE` env var to the existing `lucos_dns` image. `startup.sh` branches on it to generate either a `type primary` or a `type secondary` `named.conf.local`, and gated the primary-only `sync` container behind a `COMPOSE_PROFILES=primary` profile.
- **lucas42/lucos_configy#208** added `xwing` as a second host on the existing `lucos_dns` system entry, and — because a system with a `domain` is normally restricted to a single host — weakened the configy validation rule to apply that restriction only to systems that expose an `http_port`.

Review (lucas42/lucos_dns#79, this ADR's discussion thread) found both changes were working around a modelling gap rather than closing it, and that the env-var approach **cannot actually work**:

1. **The deploy pipeline has no per-host config.** The `lucos/deploy` orb fetches a system's production envfile from `creds.l42.eu:$CIRCLE_PROJECT_REPONAME/production/.env` — keyed on the **repository name**. Every deploy job for a repo (`deploy-avalon`, `deploy-xwing`) reads the *same* envfile. The only per-host variable the pipeline injects is `HOSTDOMAIN`. So `DNS_MODE=primary` on avalon and `DNS_MODE=secondary` on xwing cannot both be expressed from one creds namespace — the two hosts would receive identical values, leaving either no primary or no slaving.

2. **The configy change weakened a sound invariant.** The `domain ⇒ at most one host` rule exists to keep one domain name resolving to one host (configy's `/systems/subdomain/{root}` endpoint, consumed by `config-sync.py` and the router, maps one subdomain to one host). Gating it on `http_port` couples "speaks HTTP" to "may be multi-homed on a domain" — unrelated properties — and made the data assert something false: that `lucos_dns` serves `dns.l42.eu` on `xwing`, when `xwing` serves `dns2.l42.eu`.

The underlying problem is that lucos models **one system → one domain → one host**, and the homogeneous multi-host systems we already run (`lucos_firewall`, `lucos_docker_health`) fit only because they have *no* domain and run identically on every host, differentiated solely by `HOSTDOMAIN`. The DNS secondary is the estate's first *heterogeneous* multi-host case: the same software in two roles (primary/secondary), under two public hostnames (`dns.l42.eu` / `dns2.l42.eu`), needing different config per host.

## Decision

Model the primary and secondary nameservers as **two separate systems in two separate repositories**:

| System / repo | Host | Domain | Role |
|---|---|---|---|
| `lucos_dns` (existing) | `avalon` | `dns.l42.eu` | primary (authors zones via `lucos_dns_sync`) |
| `lucos_dns_secondary` (new) | `xwing` | `dns2.l42.eu` | secondary (slaves all zones from avalon via TSIG AXFR/IXFR) |

The deciding factor is the estate-wide **`repo == system` invariant**. The repository name (`CIRCLE_PROJECT_REPONAME`) is the join key for, at least:

- the creds namespace — `creds.l42.eu:$CIRCLE_PROJECT_REPONAME/production/.env`;
- monitoring suppression during deploy — `monitoring.l42.eu/suppress/$CIRCLE_PROJECT_REPONAME`;
- Loganne deploy events — `componentDeployed` / `systemDeployed`;
- `lucos_repos` convention auditing — each repo is a `RepoTypeSystem` checked against its configy entry.

Modelling two systems inside one repo would mean a system code (`lucos_dns_secondary`) with no matching repo, diverging from all of the above. Reconciling that would require threading a `system-name` override through the shared `lucos/deploy` orb (creds-fetch, monitoring-suppress, Loganne) and revisiting `lucos_repos`' repo-to-system mapping — a multi-point change to estate-critical infrastructure to benefit a single secondary nameserver. That is disproportionate. Two repos respects the invariant and requires **no shared-tooling change at all**.

Two repos is also the cleanest data model of the options considered:

- configy holds **two normal single-host systems**, each with one host and one correct domain. The existing `domain ⇒ at most one host` validation rule holds unchanged — so **lucas42/lucos_configy#208 is superseded, not reworked**: its validation change is unnecessary and its `xwing`-on-`lucos_dns` entry is wrong.
- `dns2.l42.eu` becomes a first-class configy domain, so `config-sync.py` can generate its `A`/`AAAA` glue generically from the `/systems/subdomain/l42.eu` response, removing the hardcoded `'xwing'` special-case and the `hosts[0]` fragility currently in that file.
- The secondary repo has no `DNS_MODE` branch — it simply *is* a secondary, with straight-line startup. The `TSIG_SECRET` lives in both systems' creds namespaces (same value); it was never a repository artifact, so the duplication is inherent to the design rather than a cost of splitting.

### Zone list

The one thing genuinely shared between the two servers is the set of five managed zones (`l42.eu`, `s.l42.eu`, `lukeblaney.co.uk`, `rowanblaney.co.uk`, `tfluke.uk`) — the primary authors them, the secondary must slave exactly the same set. With two repos that list is stated in both places.

**We hardcode the five zones in the `lucos_dns_secondary` repo**, and document in its README that adding or removing a managed domain is a two-repo change. This is proportionate: the set of domains lucas owns changes roughly once every few years. We deliberately do **not** add a cross-repo convention check for this — `lucos_repos` performs per-repo audits, and cross-repo content-equality is not a fit for that model; forcing it in would be the tail wagging the dog.

If the zone set ever begins to grow or churn meaningfully, the standard DNS-native answer is a **BIND catalog zone**: the primary publishes a catalog (generated by `lucos_dns_sync`, remaining the single source of truth) and the secondary slaves it, auto-configuring every member zone with no per-zone config and no duplicated list. We record this as the designated future option rather than building it now — it adds a moving part (the catalog zone) to solve a duplication problem that does not yet hurt, and it would need confirming that the BIND version shipped in the secondary's base image supports catalog zones (≥ 9.16).

### Filesystem hygiene for the secondary

Because the secondary no longer shares the primary's image, two warts identified during review disappear by construction, and the secondary repo should be built to avoid reintroducing them:

- The secondary serves every zone from BIND's slave-zone store; it must not ship the primary's static authoritative zone files (`/etc/bind/zones/`), which would sit inert and mislead an operator into thinking the box serves them locally.
- The directory BIND writes received zone transfers to is **BIND runtime state**, not generated config. It must be named honestly (e.g. `slave-zones/` / `received-zones/`, not lumped under a `generated-zones` name that elsewhere means "generated by `lucos_dns_sync`"), persisted on a volume so the secondary can serve last-known-good across a restart while the primary is down, and writable by the `named` runtime user (verify, don't assume — `startup.sh` runs as root).

## Alternatives considered

1. **Same image, env-var-selected role** (the lucas42/lucos_dns#95 approach). Rejected: the deploy pipeline has no per-host creds, so `DNS_MODE` cannot differ between avalon and xwing from one namespace. It also bakes the role into the image as a shell conditional and leaves the configy `domain`/`dns2` modelling unresolved.

2. **Same image, role derived from `HOSTDOMAIN`.** The one per-host variable the pipeline does inject is `HOSTDOMAIN`, so the image could infer "avalon ⇒ primary, else secondary". Rejected: it hardcodes the topology into the image (promotion of the secondary becomes an image change), and still leaves the configy data model wrong.

3. **Two systems in one repo.** The first-choice model during review, until the `repo == system` invariant was weighed (see Decision). Rejected: it breaks the repo-keyed creds / monitoring / Loganne / `lucos_repos` join across the estate, requiring shared-orb changes disproportionate to one secondary.

4. **Generalise configy to per-host roles** (`hosts: [{host, role, domain}]`). The most expressive model. Rejected as premature: it touches every configy consumer (router, firewall, `config-sync`, backups) to serve a two-instance case with a single driver. Recorded as the model to revisit if a *third* heterogeneous multi-host system appears.

5. **BIND catalog zones now.** Rejected as premature for five rarely-changing zones (see Zone list); designated as the future option if churn grows.

## Consequences

### Positive

- **No shared-tooling change.** The repo-keyed creds / monitoring / Loganne / `lucos_repos` machinery works unchanged for both systems. The secondary ships with today's pipeline.
- **Honest, validating configy data.** Two single-host systems with correct domains; the existing one-domain-one-host invariant is preserved rather than weakened (supersedes lucas42/lucos_configy#208).
- **`config-sync.py` simplifies.** `dns2` glue is generated generically from configy; the hardcoded `'xwing'` special-case and the `hosts[0]` ordering fragility are removed.
- **The secondary is self-evident.** A small repo that is visibly just a BIND slave — no `DNS_MODE` conditional, no inert primary-only files, no misleadingly-named directories.
- **Independent BIND patching with intact `/_info`-based monitoring.** Each system has its own `/_info`, which is the natural home for the SOA-serial-consistency check ADR-0003 called for (lucos_monitoring#189 was closed `not_planned` precisely because service-specific checks belong in the service, not the monitoring repo).

### Negative

- **The zone list is stated in two repos.** Adding or removing a managed domain is a two-repo change; a forgotten secondary update fails silently and only surfaces during a primary outage. Mitigated by documentation now and by the catalog-zone option if churn grows. This is the one genuine cost of the split.
- **Two BIND images to keep version-aligned.** Dependabot bumps each repo independently; the primary and secondary can briefly run different BIND builds. Acceptable — AXFR/TSIG is stable across versions, and a short skew in a redundancy pair is not a correctness problem.
- **A new repository to maintain.** One more repo in the estate (CI config, Dockerfile, dependabot, codeql, README). Small and largely boilerplate, but non-zero.
- **Operational correlation persists.** As ADR-0003 already noted, primary and secondary still share a human operator and similar codebase lineage; the split reduces but does not eliminate correlated failure. Standard BIND last-known-good behaviour on a bad transfer remains the backstop and must be verified during implementation.

## Follow-up actions

To be filed/triaged as separate issues (the coordinator owns labels, priority, and board placement):

- **`lucos_dns_secondary` (new repo + configy system)** — scaffold the repo (Dockerfile, CircleCI `deploy-xwing`, dependabot, codeql, README), add the `lucos_dns_secondary` configy system entry (`xwing`, `dns2.l42.eu`, `public_ports` 53/tcp+udp), and create its creds namespace. Hardcode the five slaved zones; name and persist the slave-zone store per *Filesystem hygiene* above. **[owner: lucos-system-administrator]**
- **`lucos_dns` (re-scope of lucas42/lucos_dns#79 / PR #95)** — keep only the primary-side changes: TSIG `allow-transfer` + `also-notify` to the secondary, the second `NS` record (`dns2.l42.eu`) in zone templates and static zones, and generic `dns2` glue generation in `config-sync.py` (drop the hardcoded `'xwing'` special-case). Remove all `DNS_MODE` / secondary code from this repo.
- **`lucos_configy` (close lucas42/lucos_configy#208)** — superseded: the validation change is unnecessary and the `xwing`-on-`lucos_dns` host entry is wrong under this model.
- **`TSIG_SECRET` in production creds** — written to both the `lucos_dns` and `lucos_dns_secondary` production namespaces (same value). **[lucas42 — production creds]**
- **Registrar NS + glue (lucas42/lucos#111)** — already tracked; unblocked once the secondary is live and verified.
- **SOA-serial-consistency monitoring** — re-home the ADR-0003 check (lucos_monitoring#189, closed `not_planned`) as an `/_info`-exposed signal on the DNS systems.
