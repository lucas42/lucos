# ADR-0001: User-Agent strings for inter-system HTTP requests

**Date:** 2026-03-04
**Status:** Accepted
**Discussion:** https://github.com/lucas42/lucos/issues/19

## Context

lucos is a collection of services that communicate with each other over HTTP. When debugging issues in access logs -- for example, understanding which service is responsible for unexpected traffic, diagnosing error spikes, or tracing request flows -- the User-Agent header is the most readily available signal for identifying the caller.

Historically, User-Agent strings have been inconsistent across lucos services:

- Some services set a hardcoded system name (e.g. `lucos_monitoring`).
- Some use the default User-Agent provided by their HTTP client library (e.g. `python-requests/2.31.0`, `Go-http-client/1.1`).
- Some set no User-Agent at all.

This inconsistency means that access logs often cannot answer the question "which lucos service made this request?" without cross-referencing IP addresses, timing, and other indirect signals.

Meanwhile, lucos_creds now provides a `SYSTEM` environment variable to every service (e.g. `lucos_monitoring`, `lucos_media_manager`). This variable is available in every environment (development and production) and is already used for `/_info` responses. It provides a natural, consistent identifier that does not require hardcoding per-service.

## Decision

All outbound HTTP requests from a lucos service must set the `User-Agent` header to the value of the `SYSTEM` environment variable.

This applies to:

- Service-to-service calls (e.g. an API calling lucos_loganne to emit an event).
- Monitoring fetches (e.g. lucos_monitoring polling `/_info` endpoints).
- Webhook deliveries.
- Any HTTP client library or utility function used within a service.

The raw `SYSTEM` value is used directly (e.g. `lucos_media_manager`), without appending a version number, slash, or other suffix. At lucos scale, there is exactly one instance of each service, so the system name alone is sufficient to identify the caller. The deployed version is already tracked via CI and `/_info` -- embedding it in the User-Agent would add noise for no practical debugging benefit.

Shared libraries that make HTTP requests on behalf of a service should accept the User-Agent as a parameter or read it from `SYSTEM` at initialisation, rather than hardcoding a library-specific User-Agent.

## Consequences

### Positive

- **Consistent caller identification in access logs.** Every request from a lucos service is immediately attributable to a specific system, without needing to cross-reference IP addresses or timing.
- **Minimal implementation effort.** Most HTTP client libraries accept a User-Agent parameter. Setting it to `os.environ["SYSTEM"]` (or equivalent) is typically a one-line change per client.
- **No new infrastructure required.** The `SYSTEM` variable is already provided by lucos_creds to every service. No configuration changes are needed -- only application code updates.
- **Aids debugging of unexpected traffic patterns.** If a service starts receiving unusual request volumes, the access logs immediately show which system is responsible.

### Negative

- **Requires auditing all existing services.** Each service's HTTP client usage needs to be reviewed and updated where non-conformant. This is a one-time cost, but it spans many repositories.
- **No version or instance information.** If lucos ever runs multiple instances of a service, or needs to correlate a bug with a specific deployed version, the User-Agent alone will not be sufficient. This is an acceptable trade-off at the current single-instance-per-service scale, and the convention can be extended later if needed.
- **Library defaults are lost.** Some libraries include useful metadata in their default User-Agent (e.g. client version, runtime version). This information is sacrificed for consistency. In practice, this metadata has not been useful for debugging lucos services.

### Follow-up actions

- Audit all lucos services and libraries to identify non-conformant HTTP clients (tracked in #28).
- File per-service tickets for each service that needs updating.
