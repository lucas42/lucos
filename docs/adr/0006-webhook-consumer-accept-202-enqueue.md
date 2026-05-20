# ADR-0006: Webhook consumer concurrency / burst handling

**Date:** 2026-05-20
**Status:** Accepted
**Discussion:** https://github.com/lucas42/lucos/issues/165

## Context

`lucos_loganne` is the estate's event bus. Producers (any lucos service emitting a state-change event) POST to loganne; loganne fans the event out to each registered consumer URL declared in `src/webhooks-config.json`.

Loganne's delivery model — see `src/webhooks.js#trigger` — is fire-and-forget per-event in parallel: every registered consumer URL for a given event is dispatched via an `async` callback inside `hooks.forEach(...)`, all in flight at once. A single automatic retry runs after `RETRY_DELAY_MS` (30 seconds) if the first attempt fails. There is **no concurrency cap, no per-consumer queue, and no producer-side backoff** for a slow or saturated consumer.

This means a producer that emits events at high cadence — e.g. a seinn playback batch firing `trackUpdated` at ~1 event every 2.5 seconds — can put ~150 outbound fetches in flight against the slowest consumer with no smoothing. If that consumer's webhook handler does its work synchronously inside the HTTP response cycle, individual deliveries time out, the single retry doesn't recover, and events fall to `failure` state requiring manual SRE intervention to clear.

The 2026-05-19 webhook-error-rate incident is the canonical case. Thirteen events were left stranded in loganne (nine destined for `lucos_media_weightings`), traced to a seinn-side playback batch rather than a network problem or a weightings outage. The proximate cause was that weightings processes each webhook inline, has no internal queue, and could not absorb the burst.

The post-incident architect↔SRE↔lucas42 conversation surfaced a more general question than the specific weightings fix: where in the system should burst absorption live when one producer fans out to N independent consumers, each with its own architecture, its own downstream dependencies, and its own definition of "processed"? Loganne does not — and cannot reasonably — know any of that. The natural place to absorb burst is at each consumer's receive boundary.

## Decision

**Webhook consumers absorb burst input on their own receive boundary, using the accept-202-enqueue pattern.** Each consumer's webhook endpoint:

1. Receives the POST, validates the `Authorization` header.
2. **Immediately responds `202 Accepted`** — the body has been accepted for *processing*, but processing has not yet completed. This is the correct semantic per RFC 7231 §6.3.3; `200 OK` would imply that processing is done by the time the response is returned, which under this pattern is not the case.
3. Enqueues the event onto an internal queue.
4. A worker (in-process goroutine, background thread, separate process — consumer's call) drains the queue at the consumer's own pace.

The receive path is therefore O(parse + enqueue), independent of how slow the consumer's downstream processing is. Loganne sees a fast 202 regardless of consumer backlog, and never times out a delivery.

The queue is **in-memory or persistent — the consumer chooses based on its own durability needs.** A consumer for which dropping an event on restart is a recoverable annoyance (e.g. one whose source-of-truth lives elsewhere and can be re-fetched on the next event) is fine with an in-memory channel. A consumer for which dropping is a real loss (e.g. one whose downstream writes are not idempotent or whose state cannot be reconstructed from a re-fetch) should use a persistent queue. The ADR does not mandate one over the other.

Producer-side concurrency capping is **explicitly not how this is addressed.** Loganne does not gain a per-consumer queue, a global semaphore, or any other smoothing mechanism. The reasons are in "Alternatives considered" below.

## Scope

This is **a pattern for webhook consumers**, and applies in two ways:

- **For new consumers**: the pattern is the default. New webhook-handling code in lucos services should be written as accept-202-enqueue from day one, not as inline-processing-inside-the-HTTP-response.
- **For existing consumers**: a retrofit is only undertaken where there is diagnostic evidence that the consumer is the bottleneck under burst input. The diagnostic vehicle is per-consumer instrumentation — access-log status codes plus response times — landed under whichever specific ticket flags the suspect consumer (e.g. `lucas42/lucos_media_weightings#230` is the first concrete instance of this, for weightings specifically).

There is **no estate-wide rollout** of this retrofit. The cost of rewriting a working webhook handler that is not currently a bottleneck exceeds the speculative benefit. Existing consumers stay as-is unless and until evidence puts them in the bottleneck-suspect bucket.

This restraint is deliberate: it would be very easy to read this ADR as licence to retrofit every webhook handler in the estate "for hygiene". That is not the intent. The pattern is the default for new work; existing work moves under it on evidence.

## Alternatives considered

### Producer-side concurrency cap in loganne

The most direct fix: give loganne a per-consumer semaphore or per-consumer queue, so that a slow consumer can never have more than N outbound fetches in flight from loganne's side. Bursts are smoothed at the producer.

**Rejected.** Loganne is not the right place to encode each consumer's burst tolerance. The right N for `lucos_media_weightings` is a property of weightings' downstream architecture (its DB, its writer model, its `_info` poll interval); the right N for `lucos_media_seinn` is a property of seinn's architecture; and so on. Encoding all of that in loganne's config makes loganne responsible for every consumer's internal model. That is the opposite of the loose coupling the webhook layer is meant to provide. It also means loganne gains a per-consumer knowledge dependency that gets stale as consumers evolve.

A second-order point: any producer-side cap *introduces* delivery latency for the slow consumer's events, where a fast consumer-side accept currently absorbs the same burst with near-zero added latency. The receiver-side pattern is strictly better on that axis.

### Expand loganne's retry policy

Add more retries, with exponential backoff, so transient overload at the consumer is recovered automatically without operator action.

**Rejected.** This addresses the symptom (delivery failures end up in `failure` state requiring manual clear) without addressing the cause (the consumer can't keep up with the input rate). Extra retries against a saturated consumer keep adding load to a system that is already failing to drain. Loganne's existing single 30s retry already covers the *transient* failure case (deploy window, restart blip); for *saturation* failures it cannot help.

### Dead-letter queue on the producer side

Add a DLQ to loganne so events that fail delivery after retries land somewhere the consumer can re-process when capacity returns.

**Rejected as scope creep for this ADR.** A DLQ does belong somewhere in the system, but its right home is the consumer (which knows when it has capacity again and how to re-process). The producer-side DLQ would re-create the per-consumer-knowledge problem of the producer-side concurrency cap. If a consumer wants a DLQ, the accept-202-enqueue pattern gives it the natural place to put one — between the queue and the worker.

### Per-consumer rate-limit advertised in loganne's config

The middle ground: loganne respects a `maxConcurrency` field on each consumer's registration, defaulting to unlimited. The consumer self-declares its tolerance.

**Rejected.** Less bad than a loganne-encoded cap, but still puts loganne in the middle of a property of the consumer. The consumer's own receive endpoint is a better declaration site for the same information: if a consumer can absorb arbitrary bursts via enqueue, it should; if it can't, the cap belongs in its handler, not in a registration field one repo away.

## Consequences

### Positive

- **Burst-induced delivery failures structurally disappear** for any consumer that adopts the pattern. Loganne always sees a fast 202; never times out on a slow consumer.
- **Each consumer owns its own architectural choice.** Persistent queue, in-memory channel, multiple workers, dynamic sizing — the consumer picks what fits its actual durability and throughput needs.
- **No producer-side coupling.** Loganne stays simple — fire-and-forget per-event, single retry — and gains no per-consumer knowledge. Adding a new consumer to the estate adds one URL to `webhooks-config.json` and nothing else.
- **The fix scales with the problem.** Only consumers under burst pressure pay the retrofit cost. Consumers that handle inline processing fine today don't get touched.
- **The pattern is a long-standing async-receive idiom**, not a lucos invention. New contributors recognise it. There is no novel mechanism to learn.

### Negative

- **The 202 response cannot signal a *processing* failure.** It only signals that the event has been accepted. If the worker fails on a queued event (e.g. downstream DB unreachable, downstream API returns 500), there is no HTTP response left to communicate that failure back to loganne. This is the central trade-off of the pattern and it must be addressed explicitly per consumer:
  - The consumer's own `/_info` payload should expose queue depth and/or processing-failure rate as a check, so failure-to-process is visible to monitoring.
  - The consumer's logs should carry an error line per failed event that includes enough context to recover the event (event id, predicate, payload key) — manual recovery, when needed, is then a log query rather than an inbox dive.
  - The 2026-05-19 incident is the worked example of this trade-off going the wrong way: weightings was processing inline (so the HTTP response *was* the processing signal), but the failure mode also exhausted the HTTP path. Under the new pattern weightings would have accepted the burst with no HTTP failures but with a visibly-growing internal queue and failed-process count — which is a strictly better signal to act on than 13 events stranded in loganne.
- **In-memory queues lose events on consumer restart.** A consumer that picks the in-memory option accepts that pending events at restart time are dropped. For most lucos consumers this is acceptable — the source of truth lives elsewhere and the next event refreshes the same state — but the consumer's design has to confront this consciously rather than inheriting it accidentally.
- **A queue can fill faster than the worker can drain.** Under sustained burst (rather than spike burst), the queue grows without bound. The consumer must decide on a bounded-vs-unbounded queue at design time, and what to do when bounded queues are full (reject the 202 with a 503? drop oldest? drop newest? log and continue?). The ADR does not prescribe; the consumer's `/_info` check on queue depth makes the choice observable.
- **Inline processing is occasionally simpler and the right call.** For a webhook handler whose only job is `INSERT one row, return`, accept-202-enqueue is over-engineering. The scope rule above ("only retrofit on evidence") encodes this: the pattern is a default for new code only when burst absorption is a plausible concern.

### Out of scope

- **Estate-wide retrofit of existing consumers.** Excluded by the scope rule above. Each consumer joins the pattern when evidence puts it in the bottleneck-suspect bucket, never pre-emptively.
- **Any change to loganne's delivery model.** Loganne stays fire-and-forget per-event with the existing single 30s retry, no concurrency cap. The protections raised in `lucas42/lucos_loganne#370`, `#371`, `#454`, and `#456` (failThreshold, dependsOn, single-retry) address the *alerting* and *transient-failure* sides of webhook delivery; they do not — and are not intended to — address burst handling, which lives at the receiver.
- **A specific consumer's design.** Per-consumer retrofit tickets — when they exist — pick their own queue technology, worker model, and `/_info` instrumentation. This ADR is the framing they sit inside, not a template they copy.

### Follow-up actions

- No estate-wide rollout ticket is raised. Per the scope rule.
- Per-consumer retrofit tickets are raised only on diagnostic evidence. `lucas42/lucos_media_weightings#230` is the first such candidate, currently scoped to instrumentation (access-log status codes + response times); it produces the evidence on which a weightings-side accept-202-enqueue retrofit can be raised as a separate ticket if and when the data supports it.
- This ADR is the canonical reference for the pattern. New webhook-handler PRs that follow the inline-processing shape should be expected to justify that choice in review, with reference to this document.

## References

- `lucas42/lucos_loganne/src/webhooks.js` — `trigger()` (fire-and-forget per-event parallel) and `RETRY_DELAY_MS` (30s single retry).
- `lucas42/lucos_loganne/src/webhooks-config.json` — current consumer URL registrations.
- 2026-05-19 webhook-error-rate incident — the canonical worked example for this ADR. Covered in SRE ops-check notes for the next routine.
- `lucas42/lucos_media_weightings#230` — diagnostic instrumentation (access-log status code + response time), first concrete step out of the post-incident conversation; produces the evidence that would justify a weightings-specific retrofit.
- `lucas42/lucos_media_seinn#453` and `lucas42/lucos_media_manager#262` — paired client/server work from the same incident, addressing a different layer (the producer's behaviour during failure runs, not the consumer's burst absorption).
- `lucas42/lucos_loganne#370`, `#371`, `#454`, `#456` — closed; existing protections on the alerting and transient-failure sides of loganne. Named here so it is clear they are *not* superseded by this ADR — they continue to apply and address a different problem.
