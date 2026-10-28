Agent: Copilot SDK in VS Code (AI assistant)

## Quality-attribute scenarios

1. Peak-load checkout latency  
   Source · stimulus: shopper submits order during 11.11 sale at 200 orders/s.  
   Environment: `checkoutservice` runs in Kubernetes with current p95 about 300 ms and new requirement R-12 adds MailFast, loyalty and analytics after order.  
   Response: `PlaceOrder` completes and returns to client without waiting for downstream post-order work.  
   Response measure: p95 of `PlaceOrder` stays at or below 500 ms, matching [docs/scenario.md](../../docs/scenario.md) and the ADR objective in [docs/adr/0002-2b-adr.md](../../docs/adr/0002-2b-adr.md).

2. Downstream failure isolation  
   Source · stimulus: MailFast latency p99 8 s, failure rate 0.5 %, or loyalty/analytics service unavailable.  
   Environment: production Kubernetes, with order flow and queueing under normal operation.  
   Response: order still succeeds, and post-order actions are queued and retried without blocking the customer transaction.  
   Response measure: zero order failures caused by downstream dependency outages; queue retry and DLQ handling continue until success or operational intervention, consistent with the availability requirement in [docs/scenario.md](../../docs/scenario.md).

3. Loyalty correctness under retries  
   Source · stimulus: duplicate `order.placed` messages or consumer retry after crash.  
   Environment: RabbitMQ redelivery, broker restarts, and repeated consumer execution.  
   Response: each order awards loyalty points once, within 1 minute of the order.  
   Response measure: exactly one points award per `order_id`, with no double-points bug under retry and redelivery, as stated in [docs/scenario.md](../../docs/scenario.md).

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | The ADR correctly isolates the critical path by moving MailFast, loyalty and analytics out of synchronous checkout. This matches the latency and availability goals in [docs/scenario.md](../../docs/scenario.md). | The current synchronous design would worsen p95 significantly because MailFast is slow and rate-limited; the asynchronous split directly reduces checkout contention and makes order completion independent of downstream health. |
| Risk | The ADR claims “Exactly-once delivery” for loyalty, but distributed messaging systems usually achieve “at-least-once delivery with idempotent processing,” not true exactly-once end-to-end. | The design depends on retry/redelivery semantics; without a stronger contract for deduplication and transactional state, duplicate points can still happen under broker failures or consumer restarts. |
| Sensitivity point | The email path is highly sensitive to MailFast’s p95/p99 and 50 emails/s rate limit. | If the email consumer is not strictly rate-limited and retried with backoff, queue lag can grow quickly and the whole post-order backlog will be delayed; this is the main throughput bottleneck. |
| Trade-off point | Durable queues, publisher confirms, DLQ, and retries improve reliability but increase operational complexity and cost. | The architecture gains better availability and fault isolation, but it also requires monitoring, replay, retention, and maintenance of RabbitMQ and the worker ecosystem. |

## Missing or weak

- The ADR resolves “Outbox Pattern” as a decision but does not specify the actual data model or publication contract. Without a concrete outbox table, commit order, and worker responsibility, the system may still lose events between the order commit and message publish. This is a major gap in a design that relies on “not losing events.”
- The ADR discusses DLQ and retry, but it does not state how the system replays or recovers from messages that fail after 5 retries, nor how queue depth and DLQ alerts will be operationalized. In a business-critical workflow, recovery is part of the architecture, not an afterthought.
- The option set is narrow: it compares synchronous calls against RabbitMQ and workflow orchestrators, but does not consider a simpler background-worker pattern or a per-service async queue model. The chosen option is strong, but the ADR should justify why a broker-based design is preferable to a lighter-weight internal worker for this specific team and scale.
- The consequences section is incomplete; it mentions operational burden but not the cost of RabbitMQ operations, monitoring, and replay tooling. For a 4-developer team, this matters a lot for maintainability and operability.
- A key claim is likely overstated: “Exactly-once processing” in the loyalty consumer is not a safe statement unless the service enforces a unique constraint and transactionally checks/updates state. The ADR should say “effectively-once” or “at-least-once with idempotency,” with the idempotency key and DB contract stated explicitly.

## Verdict

Accept with changes.

Three most important changes:
1. Replace the “exactly-once” wording with an explicit at-least-once + idempotent consumer contract, including the unique constraint and DB transaction pattern for the loyalty service.
2. Specify the transactional outbox flow in detail: how the order transaction commits, how the outbox row is published, and what happens if RabbitMQ confirm fails or the worker crashes.
3. Add a concrete operational plan for queue growth, DLQ replay, and backlog alerts; otherwise the queue may become the new single point of failure for post-order processing.
