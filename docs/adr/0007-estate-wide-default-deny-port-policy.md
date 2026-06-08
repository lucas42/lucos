# ADR-0007: Estate-wide default-deny port policy

**Date:** 2026-05-22
**Status:** Accepted
**Amended:** 2026-06-01 — per-host enforce control (see Amendment 1 at end)
**Amended:** 2026-06-08 — DOCKER-USER bridge-interface RETURN rules (see Amendment 2 at end)
**Discussion:** https://github.com/lucas42/lucos/issues/169

## Context

The lucos estate has no host-level firewall. Docker's iptables rules forward published-container traffic from the public interface into the container, but those rules are *routing*, not *access control*. The consequence is that **every port that any container publishes (via `ports:`), or that any process running with `network_mode: host` binds, is immediately internet-reachable.** No review gate. No approve-before-expose. No default-deny.

This has been a known architectural property — for example, `http://178.32.218.44:8019/_info` (loganne's backend port on avalon) returns a 200 to any internet client, bypassing the nginx router entirely. The router provides TLS termination and domain routing; it does not provide access control. Application-level auth (`CLIENT_KEYS`) is the only real protection.

The consequences turned out to be broader than that observation suggested. The 2026-05-21 finding on `lucos_monitoring`'s Erlang Port Mapper Daemon (port 4369) — internet-reachable for ~13 months, accumulating daily portscan attempts in the container logs — was the trigger. A wider port survey on avalon found:

| Port | Service | Source | Risk |
|---|---|---|---|
| 4369 | Erlang EPMD | `lucos_monitoring` (`network_mode: host`) | Node-directory service; node-name enumeration |
| 34031 | Erlang distribution | `lucos_monitoring` (`network_mode: host`) | The actual inter-node protocol; cookie-gated RCE if cookie leaks |
| 953 | BIND RNDC | `lucos_dns_bind` (`0.0.0.0:953` publish) | DNS control channel; defended at app-level by TSIG + `localnets` ACL, but the protocol surface is unnecessarily exposed |
| 5355 | LLMNR | systemd-resolved (host OS) | Link-local name resolution, low practical risk |

None of these are actively exploitable today (the Erlang VM correctly rejects the malformed handshakes; RNDC's ACL likely rejects external sources at the protocol level), but the exposure is a structural gap. The next infrastructure port that an agent or developer accidentally binds to `0.0.0.0` becomes the same problem.

A second discovery during scoping: I'd been calling `lucos_time` an NTP service based on its name. It isn't — it's a Node.js HTTP service. The actual reason it (and `lucos_monitoring`) used `network_mode: host` was IPv6 outbound (commits `2d36838` and `ab9f6b9`, both 2024-04-29) — Docker's default bridge network was IPv4-only at the time and host networking was the path of least resistance to give the container access to the host's IPv6 stack, needed to reach IPv6-only services such as those on `salvare`. The original IPv6 requirement still applies, but Docker Compose has supported per-network IPv6 since 20.10+, so host mode is no longer the only path.

## Decision

Adopt a three-layer model. Each layer fails safe; each layer assumes the others might not exist.

1. **Layer 1 — Application auth (`CLIENT_KEYS`, TLS, TSIG, etc.).** Already in place across the service band. Unchanged by this ADR.
2. **Layer 2 — Host-level default-deny firewall.** The new layer. Default DROP on inbound; explicit allow-list per port; rules generated from `lucos_configy` so they cannot go stale relative to deployed services.
3. **Layer 3 — Provider-level firewall.** Not adopted. Explicitly out of scope per "Alternatives considered".

The Layer 2 firewall has three properties:

- **Rules derive from a single source of truth (`lucos_configy`).** Each service that needs public network exposure declares it in configy via a new `public_ports` field (alongside `http_port` and `hosts`). A port is publicly reachable on a host **if and only if** it is declared as a `public_ports` entry on a service whose `hosts:` list contains that host. There is no other allow-list. Removing a service from configy closes the port on the next firewall regen. The default state for any port not declared is closed.
- **The firewall runs in a container, not on the host.** A new `lucos_firewall` service — single-purpose container with `cap_add: [NET_ADMIN]` and `network_mode: host` (legitimate use: it needs the host's netfilter to apply rules) — fetches configy, generates an iptables ruleset, and applies it via `iptables-restore`. It re-applies on configy changes via a poll loop. The only host-side configuration is the Docker daemon's IPv6 enablement; everything else lives in container images and configy.
- **`network_mode: host` is prohibited by default.** Existing exceptions (`lucos_monitoring`, `lucos_time`) migrate to bridge networking with IPv6 enabled at the compose-network level. New services adding `network_mode: host` require written justification in their `docker-compose.yml` and a corresponding architect review.

### Specifics

**`public_ports` schema in configy.** Inline list under each service:

```yaml
lucos_router:
    hosts: [avalon, xwing]
    public_ports:
        - { port: 80,  protocol: tcp, purpose: "HTTP (Let's Encrypt redirect to HTTPS)" }
        - { port: 443, protocol: tcp, purpose: "HTTPS — TLS termination + domain routing" }
lucos_mail:
    domain: mail.l42.eu
    http_port: 8022
    hosts: [avalon]
    public_ports:
        - { port: 25, protocol: tcp, purpose: "SMTP inbound" }
lucos_dns:
    domain: dns.l42.eu
    hosts: [avalon]
    public_ports:
        - { port: 53, protocol: tcp, purpose: "DNS (TCP fallback)" }
        - { port: 53, protocol: udp, purpose: "DNS (primary)" }
lucos_creds:
    domain: creds.l42.eu
    http_port: 8031
    hosts: [avalon]
    public_ports:
        - { port: 2202, protocol: tcp, purpose: "SSH endpoint for creds read/write" }
lucos_locations:
    domain: locations.l42.eu
    http_port: 8028
    hosts: [avalon]
    public_ports:
        - { port: 8883, protocol: tcp, purpose: "MQTT — anonymous-disabled, TLS + password" }
```

**Note that `lucos_router` declares 80/443 as `public_ports`, not the firewall.** The firewall has no hardcoded base allow-list of "service" ports — every public port is declared as `public_ports` on the service that uses it. The only entries the firewall hardcodes are the things that don't belong to any lucos service: SSH to the host (port 22), ICMP, loopback, and ESTABLISHED/RELATED connection tracking.

**Firewall ruleset shape.** Generated per host. Skeleton:

```
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:DOCKER-USER - [0:0]

# Loopback + connection tracking
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Host SSH (not a lucos service; hardcoded)
-A INPUT -p tcp --dport 22 -j ACCEPT

# Conservative ICMP
-A INPUT -p icmp --icmp-type echo-request -j ACCEPT
-A INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
-A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
-A INPUT -p icmp --icmp-type parameter-problem -j ACCEPT

# Per-service public_ports (generated from configy, one rule per declared port)
-A INPUT -p tcp --dport 80 -j ACCEPT    # lucos_router
-A INPUT -p tcp --dport 443 -j ACCEPT   # lucos_router
-A INPUT -p tcp --dport 25 -j ACCEPT    # lucos_mail (avalon only)
...

# Mirror the same allow-list into DOCKER-USER so Docker-published ports are filtered too
-A DOCKER-USER -p tcp --dport 80 -j ACCEPT
...
-A DOCKER-USER -j DROP

COMMIT
```

Both `INPUT` and `DOCKER-USER` are populated. `INPUT` catches host-network containers and host-process listeners. `DOCKER-USER` is the chain Docker explicitly leaves alone for user rules and catches Docker-forwarded (port-published) traffic. Docker's own chains (`DOCKER`, `DOCKER-ISOLATION-*`) are not touched.

The `FORWARD` policy is **ACCEPT**, not `DROP`: Docker owns the `FORWARD` chain and depends on forwarding being open for container traffic, so default-deny for Docker-published ports is enforced by the terminal `DROP` in `DOCKER-USER` — not by the `FORWARD` policy. The estate's default-deny therefore lives in two places (the `INPUT` policy `DROP` for host-network/host-process listeners, and the `DOCKER-USER` terminal `DROP` for Docker-published ports), never in `FORWARD`. _(An earlier revision of this skeleton mistakenly showed `:FORWARD DROP`; corrected 2026-06-08 to match the shipped `lucos_firewall` implementation.)_

**Host placement.** `lucos_firewall` runs on `avalon`, `xwing`, and `salvare` — the three active internet-facing hosts. `salvare` has no `lucos_router` deployment, so its allow-list will not include 80/443 unless something else declares them; the firewall there is correspondingly tighter.

## Scope

This ADR covers public-network firewalling. It does not cover:

- Inter-container access controls within a Docker Compose stack (those remain on the existing Compose-network model).
- Application-level authentication on individual endpoints (Layer 1, unchanged).
- Outbound firewalling (everything outbound stays accepted — `OUTPUT ACCEPT`).
- IPv6 firewalling (handled by a parallel `ip6tables` ruleset generated alongside the IPv4 one; same allow-list, same source-of-truth — implementation detail of `lucos_firewall`, not a separate ADR).

## Alternatives considered

### Hand-maintained allow-list

A static file checked into version control. Sysadmin edits it whenever a new service is added.

**Rejected.** The whole reason this ADR exists is that an exposure landed without anyone updating a list. A hand-maintained list is the most familiar shape but it has the failure mode the design is trying to prevent. Tying the firewall to configy (which is already the source of truth for deployed services) means the firewall cannot go stale relative to the deployed estate by construction.

### `ufw` (Uncomplicated Firewall)

Familiar Debian-style frontend over iptables.

**Rejected.** `ufw` is famously broken with Docker: it does not intercept Docker-published ports without third-party patches (`ufw-docker` and similar). The "I thought I closed that port but Docker re-opened it" story is well-known. The whole point of this exercise is to remove surprises; introducing a tool that has Docker-interaction surprises baked in is a backwards step.

### Native nftables instead of iptables

`nftables` has cleaner syntax and is the kernel's modern path. The shell tooling on modern Debian is `iptables-nft` (iptables CLI, nftables backend), which we could replace with `nft` directly.

**Rejected (for now).** Docker still generates iptables rules at the iptables CLI level. Mixing native `nft` rules with Docker's iptables-managed rules creates ordering and chain-naming complexity that isn't worth the syntax win at our scale. The `DOCKER-USER` chain contract is stable and works equally well via the iptables CLI. If Docker's own tooling moves natively to `nft`, this ADR can be revisited; until then `iptables-restore` is the right tool.

### OVH (or other provider) network firewall as Layer 3

The hosting provider exposes a stateless IPv4 packet filter ahead of the host network stack. Cannot be bypassed by any in-host misconfiguration.

**Rejected.** Stateless, IPv4-only, requires console access for changes, no observability from inside the host. Of all the candidates, it has the highest stale-config risk because changes require a different management surface entirely. Given a working Layer 2, the marginal benefit is low. lucas42 confirmed dropping it on 2026-05-21.

### Port-band allow-list in configy

Allow `lucos_configy` to declare a "service port band" (e.g. 8000–8050) that's allowed by default, so individual services don't need `public_ports` entries.

**Rejected.** This would treat `http_port` services (which are router backends and should NOT be publicly reachable) as legitimately public, undoing the architectural property the design is meant to enforce. The router is the front door for HTTP; backend ports are internal. The per-service `public_ports` shape forces every legitimately-public port to be declared explicitly and avoids the trap of "the 8000 band looks like it should be open".

## Consequences

### Positive

- **The "default-public" failure mode disappears by construction.** A port is reachable iff it's declared in configy. Adding a new service or changing one cannot accidentally open a port.
- **The EPMD/34031 exposure on `lucos_monitoring` is closed twice.** Once by the `network_mode: host` migration (the ports are no longer publicly bound at all), and again by the firewall's default-deny (even if a future container does bind 4369 on `0.0.0.0`, the firewall won't allow it).
- **The `lucos_dns_bind` 953 RNDC publish is closed twice.** Once by fixing the publish at source (`0.0.0.0:953` → `127.0.0.1:953`), once by the firewall (953 is not a `public_ports` entry).
- **One source of truth.** `lucos_configy` already maps `service → host → port` for the router. The firewall reads the same data. Operators don't have to keep two lists in sync.
- **No host-level state to manage.** The firewall lives in a container; rules are re-applied from configy on each container start. Reboots, image upgrades, and configy changes converge on the same canonical state.
- **`network_mode: host` becomes the documented exception**, not the default-of-convenience. Existing usages get migrated; new ones get architect review.

### Negative

- **Sub-second open window at host boot.** Between Docker daemon ready and `lucos_firewall` container ready, the host is unfiltered. Acceptable trade-off; if we ever need to harden it we can add an `iptables-restore` of a baked-in fallback ruleset to the host's boot sequence, but that re-introduces a host-config dependency this ADR is trying to avoid.
- **Configy is now a hard dependency of network reachability.** If configy is unreachable when the firewall container starts, the container fails to come up. Mitigation: ship a baked-in fallback ruleset in the firewall image (base list + nothing else, fails closed-ish). The fallback is documented inside the firewall repo, not duplicated here.
- **IPv6-on-bridge-network is a new bit of Docker config to maintain.** Daemon-side `"ipv6": true` and `fixed-cidr-v6` need to be set on each host; per-stack `enable_ipv6` must be set wherever a container needs IPv6 outbound (currently `lucos_monitoring` and `lucos_time`). This is a real ongoing cost — every new service that needs outbound IPv6 has to remember to enable it on its compose network.
- **Some existing inter-service traffic that currently traverses public IPs will hit the firewall.** lucos services already use public HTTPS endpoints for inter-service comms (e.g. `lucos_arachne_ingestor` calls `https://contacts.l42.eu`, not `http://localhost:8013`) — so the router's 443 is the entry point and there's no expected breakage. But: any service inadvertently calling a backend port directly (e.g. `http://178.32.218.44:8019` to loganne) will start failing after the firewall flips to enforce. Dry-run mode (log-only) on avalon for ~a week before enforce is the safety net for this.
- **`lucos_firewall` is a new service with operational cost.** Healthcheck, monitoring, deploy pipeline, the usual lucos things. The container's role is small but the surface area of "another thing to keep running" is real.

### Out of scope

- **Outbound firewalling.** `OUTPUT ACCEPT`. Per-service outbound restrictions would be a different ADR with a different cost/benefit balance.
- **Estate-wide host-firewall on non-lucos-managed hosts.** This ADR only covers `avalon`, `xwing`, `salvare` (the active internet-facing hosts).
- **OVH provider-level firewall.** Dropped above.
- **Replacing existing app-level auth.** `CLIENT_KEYS`, TLS, TSIG, MQTT password auth all stay. The firewall is additional, not a replacement.

### Follow-up actions

Sequenced so the most-impactful and lowest-risk changes go first. Each is a separate ticket in the relevant repo; this ADR is the umbrella.

1. **`lucos_monitoring` migrates off `network_mode: host`.** Drop host mode, enable IPv6 on the compose default network. Eliminates the EPMD/34031 exposure regardless of whether the firewall is in place. **Done first, independent of the firewall.**
2. **`lucos_time` migrates off `network_mode: host`.** Same change, same independent timeline. Both 1 and 2 should land before the firewall service is built, per the "more time to pivot if they hit problems" steer.
3. **`/etc/docker/daemon.json` IPv6 enablement on avalon, xwing, salvare.** Already in place on all three hosts — verified by sysadmin 2026-05-22. No implementation ticket needed. **However**: salvare's `fixed-cidr-v6` is set to `fe80::/64` (link-local), not a globally-routable prefix. Containers on salvare's bridge network therefore won't have internet-routable IPv6, which would silently break any future migration of a host-network IPv6-dependent container to salvare. salvare is not in scope for ADR-0007's primary migration (`lucos_monitoring` is avalon-only), so this doesn't block the work — but it is a latent landmine. Tracked as `lucas42/lucos#179`.
4. **`lucos_configy` schema adds `public_ports` field**, plus a `GET /systems/host/{host}/public-ports` (or equivalent) endpoint that returns the per-host declared port list.
5. **`lucos_configy` config populates `public_ports`** for `lucos_router` (80/443), `lucos_mail` (25), `lucos_dns` (53 TCP+UDP), `lucos_creds` (2202), `lucos_locations` (8883). Can land in the same PR as the schema change or separately.
6. **New repo `lucos_firewall`.** Generates iptables/ip6tables rules from configy; applies via `iptables-restore` / `ip6tables-restore`; runs as a container with `cap_add: [NET_ADMIN]` and `network_mode: host`. Watch loop re-applies on configy changes. Ships a baked-in fallback ruleset for the configy-unreachable case.
7. **Deploy `lucos_firewall` to avalon in dry-run mode.** Log what would be denied; do not enforce. Run for ~a week.
8. **Flip avalon to enforce.**
9. **Fix `lucos_dns_bind` RNDC publish at source.** `0.0.0.0:953` → `127.0.0.1:953` in `docker-compose.yml`. Closes the 953 exposure independently of the firewall.
10. **Deploy `lucos_firewall` to xwing.**
11. **Deploy `lucos_firewall` to salvare.**

## References

- `lucas42/lucos/issues/169` — full design conversation, including the original SRE finding, the security-team scoping, and the four-tweak sign-off from lucas42.
- `lucas42/lucos_monitoring` commit `ab9f6b9` (2024-04-29) — original addition of `network_mode: host` for IPv6 support.
- `lucas42/lucos_time` commit `2d36838` (2024-04-29) — same.
- `lucas42/lucos_configy` `README.md` — current `http_port` / `hosts` / `domain` schema; `public_ports` is the new field this ADR depends on.

## Amendment 1 (2026-06-01) — per-host enforce control

### Context for the amendment

The body above describes the firewall reading `public_ports` from configy and progressing from "dry-run" to "enforce", but is silent on *how* enforce mode is controlled per host. The implicit assumption was a single `DRY_RUN` environment variable on the `lucos_firewall` container.

Two things broke that assumption:

1. **The rollout was re-sequenced** to enforce `xwing → salvare → avalon`, flipping each host to enforce **independently and at different times**, each only after its own ≥7-day dry-run is reviewed clean (see `lucas42/lucos#182`). The original follow-up actions 7–11 (avalon-first, single-host dry-run) are superseded by that reorder; `lucas42/lucos#182` is the authoritative rollout sequence.
2. **A single env var cannot express per-host mode.** `DRY_RUN` is populated from `lucos_creds`, which is keyed by `(system, environment)` — *host* is not a dimension — and the same `docker-compose.yml` deploys to all three hosts. So an env-borne flag is necessarily identical on xwing, salvare and avalon; flipping it flips all three at once, which the staged rollout cannot tolerate.

Full options analysis: architect assessment on `lucas42/lucos#182` (2026-06-01).

### Decision

**Per-host enforce mode is a per-host fact in `lucos_configy`, not an environment variable.**

- **Source of truth.** A new per-host boolean `firewall_enforce` on the configy `Host` model (`config/hosts.yaml`), defaulting to **`false` = dry-run (log-only)**. A host enforces if and only if it explicitly declares `firewall_enforce: true`. Absence ⇒ the safe state. (Implemented in `lucas42/lucos_configy#203`, which also adds a `/hosts/{host}` endpoint exposing the host record.)
- **Identity vs. mode.** The firewall already derives its host identity from `HOSTDOMAIN` (injected per-host by the deploy orb; first DNS label → configy host key) and already polls configy per-host for `public_ports`. It reads `firewall_enforce` for the same host key from the same source. `HOSTDOMAIN` supplies *identity* (static, deploy-time); configy supplies *mode* (per-host, changeable at runtime). The two compose — neither alone is sufficient. (Implemented in `lucas42/lucos_firewall#9`.)
- **Flipping a host** is a one-line, version-controlled, PR-reviewed edit to `hosts.yaml`, picked up by the firewall's existing configy poll loop within one interval. No redeploy. The shipped timed auto-rollback remains the safety net on the enforce transition.
- **The `DRY_RUN` env var is retained only as a transition fallback and local-dev override**, subordinate to the configy value once present.

This keeps the firewall's mode and its port list reading from the **same single source of truth** (configy), consistent with the ADR's core principle that firewall state cannot go stale relative to the deployed estate.

### Fail-safe contract (configy ↔ firewall)

Because mode now also derives from configy, a configy fault must never *change* a host's behaviour into a riskier state:

- **Hold last-known-good across a configy outage.** On a failed poll, the firewall keeps applying the last successfully-fetched `(mode, ports)` pair. It does **not** re-derive mode from a failed fetch. (Composes with the existing configy-unreachable auto-rollback in the firewall.)
- **Cold start with configy unreachable and no cache ⇒ dry-run (log-only).** When mode is genuinely unknown, default to the safe state. The firewall must never enforce a mode it couldn't read against a port list it also couldn't read — that is the partial-ruleset lockout path. Unknown ⇒ safest.

### Consequences

**Positive**

- Per-host flips are reviewable, auditable, version-controlled config edits — no redeploy, no per-host env injection, no second source of truth.
- Mode and port list share one source of truth (configy), already a hard runtime dependency of the firewall; no new dependency class is introduced.
- The default (`firewall_enforce` absent ⇒ dry-run) is fail-safe: a forgotten or mistyped flag leaves a host logging-only, never locked out.

**Negative**

- Enforce mode is now coupled to configy availability as well as the port list. The fail-safe contract above bounds the blast radius, but it does mean a host's *intended* enforce posture is only realised while configy is reachable (mitigated by last-known-good caching).
- A host that should be enforcing but whose flag was never set will sit silently in dry-run. This is observable (it logs would-deny lines and never applies them) and is caught by the rollout's per-host enforce-verification step, but it is a quiet failure mode rather than a loud one.

### Superseded

- Follow-up actions 7, 8, 10 and 11 in the body (avalon-first dry-run then per-host enforce deploys) are superseded by the parallel-dry-run-then-`xwing → salvare → avalon`-enforce sequence in `lucas42/lucos#182`.

### Amendment 1 references

- `lucas42/lucos#182` — progressive deployment ticket; authoritative rollout sequence and the architect assessment that produced this amendment.
- `lucas42/lucos_configy#203` — `firewall_enforce` field + `/hosts/{host}` endpoint.
- `lucas42/lucos_firewall#9` — firewall reads enforce-mode from configy, with the fail-safe contract above.

## Amendment 2 (2026-06-08) — DOCKER-USER bridge-interface RETURN rules

### Context for the amendment

The ADR's Scope section states: _"Inter-container access controls within a Docker Compose stack (those remain on the existing Compose-network model)."_ However the original DOCKER-USER chain implementation did not actually enforce this exclusion — it simply terminated with a DROP rule, with no differentiation between external-origin traffic and Docker bridge traffic.

This created a gap on hosts where the `br_netfilter` kernel module is loaded and `bridge-nf-call-iptables=1`. On such hosts, same-bridge inter-container traffic traverses the iptables FORWARD chain, which DOCKER-USER is wired into. The terminal DROP in DOCKER-USER would therefore drop connections between containers on the same Compose-project network — e.g. `lucos_photos_api → lucos_photos_postgres` — regardless of the scope exclusion the ADR intended.

A per-host sysctl check confirmed:

| Host | `br_netfilter` loaded | `bridge-nf-call-iptables` | ICC through iptables |
|---|---|---|---|
| avalon | **yes** | **1** | **yes** |
| xwing | no | (key absent) | no |
| salvare | no | (key absent) | no |

The risk is confirmed on avalon, which is the last host to enforce under `lucas42/lucos#182`. The fix must land before avalon's enforce flip. Tracked as `lucas42/lucos_firewall#13`.

### Decision

**Add `-i br+ -j RETURN` and `-i docker0 -j RETURN` rules to the DOCKER-USER chain, positioned after the ESTABLISHED/RELATED ACCEPT and before the public-port ACCEPT rules.**

The `-i` flag in iptables matches the **ingress interface** — the interface a packet was *received on* by the host kernel. For any container-originated traffic, this is always the **source** container's bridge, regardless of where the destination container lives. A packet from stack-A (`br-aaaa`) destined for stack-B (`br-bbbb`) is received by the kernel on `br-aaaa`, and `-i br-aaaa` matches `-i br+`. The RETURN therefore fires for **all** bridge-origin inter-container traffic: same-stack (same Compose project) and cross-stack (different Compose projects on the same host) alike.

Only packets arriving from a non-bridge interface — a public-facing `eth0`, `bond0`, etc. — fall through to the public-port allow-list and the terminal DROP.

The `-i br+` wildcard matches any iptables interface name beginning with `br` — this covers the `br-xxxxxxxx` naming pattern used by Docker Compose for custom bridge networks. The `-i docker0` rule covers the default Docker bridge explicitly (its name does not match `br+`).

Updated DOCKER-USER chain shape:

```
# DOCKER-USER: external-origin traffic only — bridge ICC is returned to FORWARD chain
-A DOCKER-USER -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
-A DOCKER-USER -i br+ -j RETURN
-A DOCKER-USER -i docker0 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j ACCEPT   # per-service public ports
-A DOCKER-USER -p tcp --dport 443 -j ACCEPT
...
-A DOCKER-USER -j DROP
```

This pattern applies to both the IPv4 (`iptables`) and IPv6 (`ip6tables`) rulesets generated by `lucos_firewall`.

### Consequences

**Positive**

- All bridge-origin inter-container traffic — both same-stack (same Compose project) and cross-stack (different Compose projects on the same host) — is returned to Docker's own FORWARD rules and is not dropped by `lucos_firewall`. The host firewall polices external-origin traffic only.
- The fix is a no-op on hosts where `br_netfilter` is not loaded (xwing, salvare) — RETURN rules match bridge-interface traffic that Docker already bypasses iptables for, so no behaviour change on those hosts.
- No Compose stack networking changes required; no redeploy of application services.

**Negative / scope note**

- The exemption is intentionally broader than ADR-0007's original Scope wording ("within a Docker Compose stack"). It also covers cross-stack ICC. This is the accepted design (signed off by lucas42, 2026-06-08) for two reasons: (a) there is no trusted internal network on the lucos estate — see Layer 1 and the identity-not-from-topology principle (`lucas42/lucos#132`); (b) expressing same-stack-only exemption via `-i` alone would require fragile per-bridge rules (`-i br-aaaa -o br-aaaa`) that break whenever a stack is recreated and gets a new bridge ID. The security model for cross-stack traffic relies on application-level auth (Layer 1), not host-firewall rules.

### Amendment 2 references

- `lucas42/lucos_firewall#13` — issue tracking the blast-radius analysis and implementation.
- `lucas42/lucos_firewall#14` — PR shipping the RETURN rules and regression tests.
