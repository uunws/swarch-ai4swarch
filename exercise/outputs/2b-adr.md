Agent: Claude Code (VS Code extension), model Claude Opus 5.5 (`claude-opus-5-5`). Run b (context: `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`).

# ADR-0002: Publish an OrderPlaced event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Not given (see Open questions)

## Context and problem

R-12 (`docs/scenario.md`): after an order is placed, Online Boutique must (1) send the confirmation email through **MailFast**, replacing the mock `emailservice` logic; (2) award **loyalty points** through a new `loyaltyservice`; (3) record an **"order placed" analytics event**. Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

How it works today:

- `checkoutservice.PlaceOrder` (Go) gets the cart, prices items, quotes shipping, then calls `chargeCard` → `shipOrder` → `emptyUserCart` → `sendOrderConfirmation` **synchronously over gRPC, in sequence** (`src/checkoutservice/main.go`, `PlaceOrder`, l.230–280).
- If the email call fails, checkout only logs a warning and the order still succeeds (`main.go` l.273–277).
- The order is **not persisted anywhere**: the order ID is an in-memory `uuid.NewUUID()` (l.233), and `checkoutService` holds only gRPC client connections, with no database (l.66–86). So a "transactional outbox" in checkout's own database isn't possible without adding a store.
- The `OrderResult` sent to email contains order ID, tracking ID, shipping cost, address and items, but **no user ID and no order total** (l.265–271; the total is computed locally at l.243–250). `loyaltyservice` needs both.

Facts from `docs/scenario.md` that drive the decision:

| Topic | Fact | Consequence for this design |
|---|---|---|
| Checkout latency | p95 `PlaceOrder` ≤ **500 ms**, today approximately **300 ms** | About 200 ms of budget left. MailFast's p95 alone is **1.5 s**, so it **cannot** be called inside checkout |
| Peak load | **200 orders/s** (11.11 sale) | Post-order consumers must absorb 200 events/s |
| MailFast | p95 1.5 s, p99 8 s, approximately 0.5 % failures, **50 emails/s** limit | At peak, emails arrive 4× faster than MailFast accepts: the backlog grows by 150 emails/s (9,000 per minute of peak). Emails **must** be queued and throttled |
| Loyalty points | within **1 minute**, **exactly once** | Loyalty must not wait behind the email backlog; duplicates must be removed |
| Availability | an order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the checkout response |
| Team | 4 devs, strong in Go and gRPC; **two have used RabbitMQ** | Favors RabbitMQ over an unfamiliar broker |
| Operations | Kubernetes; prefer **open-source, self-hosted** | Rules out cloud-only managed queues (e.g. a cloud pub/sub service) |

## Decision drivers

1. **Availability of checkout:** an order never fails because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 stays ≤ 500 ms at 200 orders/s.
3. **Correctness of loyalty points:** awarded exactly once, within 1 minute.
4. **Modifiability:** next year's actions (SMS, fraud scoring, recommendations) are added without changing `checkoutservice`.
5. **Respect MailFast's 50 emails/s limit** at 200 orders/s without losing emails.
6. **Cost and operability:** open-source, self-hosted on Kubernetes, and operable by a 4-person Go team.

## Considered options

1. **Extend the synchronous chain:** `checkoutservice` calls MailFast (via `emailservice`), `loyaltyservice` and analytics over gRPC inside `PlaceOrder`, as it does today.
2. **Fire-and-forget goroutines in `checkoutservice`:** reply to the customer, then make the three calls in background goroutines with in-process retries.
3. **Publish an `OrderPlaced` event to RabbitMQ (self-hosted, in-cluster).** A topic/fanout exchange routes it to one durable queue per consumer: `email`, `loyalty`, `analytics`.
4. **Publish an `OrderPlaced` event to Apache Kafka (self-hosted, in-cluster).** One topic, with one consumer group per action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous chain | No new infrastructure; matches today's code style | MailFast p95 1.5 s / p99 8 s breaks the 500 ms budget; 50 emails/s limit vs 200 orders/s means calls are throttled or rejected during checkout; a `loyaltyservice` outage either fails orders or silently loses points; every new action edits `PlaceOrder` |
| 2. Goroutines in checkout | Keeps latency low; no new infrastructure | Work in memory is lost when a pod restarts or is scaled down (no order store, `main.go` l.66–86); no durable backlog for the 150 emails/s overflow; can't guarantee loyalty within 1 min after a crash; still coupled: each new action changes checkout |
| 3. RabbitMQ | Durable, per-consumer queues: email backlog doesn't delay loyalty; consumer prefetch and a rate limiter fit MailFast's 50/s; open-source and self-hosted (Kubernetes operator available); **two team members already know it**; new actions = new queue bound to the exchange | New stateful component to run (cluster, upgrades, monitoring); at-least-once delivery, so consumers must deduplicate; messages are removed once consumed, so analytics can't replay old events |
| 4. Kafka | Very high throughput; retained log lets analytics and future consumers (e.g. recommendations update) replay history; open-source | Heavier to operate (brokers, partitions, Strimzi/KRaft) for 200 msg/s; **nobody on the team has used it**; per-message rate limiting and retry/dead-letter handling must be built by hand; also at-least-once by default |

## Decision

Choose **option 3: `checkoutservice` publishes an `OrderPlaced` event to an in-cluster RabbitMQ** after payment and shipping succeed. Option 1 fails drivers 1, 2 and 5 outright. Option 2 cannot meet driver 3 because checkout keeps no durable state. Between 3 and 4, both meet the drivers. RabbitMQ wins on team skills and operability. Kafka's replay advantage isn't required by any current driver.

Design:

1. **Publish.** In `PlaceOrder`, replace the `sendOrderConfirmation` gRPC call (`main.go` l.273) with a publish of `OrderPlaced {order_id, user_id, email, items, total, shipping_cost, address, tracking_id, placed_at}` to exchange `orders`. Use publisher confirms with a short timeout (e.g. ≤ 50 ms, to be tuned by load test). If the publish fails, **the order still succeeds** (driver 1): log the failure, retry in the background, and alert. This keeps today's "log and continue" behaviour (l.274) but makes the gap visible.
2. **Queues.** Durable **quorum queues** (replicated across 3 RabbitMQ nodes), one per consumer: `email`, `loyalty`, `analytics`. Each has a dead-letter queue.
3. **Email consumer** (in `emailservice`, Go replaces the mock logic). Sends through MailFast with a **global limit of 50 emails/s** across all replicas. Retries the approximately 0.5 % failures with exponential backoff; permanent failures go to the dead-letter queue.
4. **`loyaltyservice`** (new, Go). Consumes from its own queue, so it isn't delayed by the email backlog (driver 3). **Exactly once = at-least-once delivery + idempotent processing:** in the same database transaction, insert `order_id` into a `processed_orders` table with a unique constraint and add the points. A duplicate delivery hits the constraint and is acknowledged without adding points.
5. **Analytics consumer.** Records the event for marketing. Where it is stored is an Open question.
6. **Future actions** (SMS, fraud scoring, recommendations) bind a new queue to the `orders` exchange: no change to `checkoutservice`.

## Consequences

- **Positive:** `PlaceOrder` no longer waits for email (MailFast p95 1.5 s); it only adds a broker publish. Expected to stay well under 500 ms; to be confirmed by load test (quality attribute: **performance**).
- **Positive:** an outage of MailFast, `loyaltyservice` or analytics only grows a queue; orders keep succeeding (quality attribute: **availability**).
- **Positive:** the email backlog at 11.11 is absorbed by the queue and drained at 50/s instead of being dropped (quality attributes: **reliability**, **scalability**).
- **Positive:** new post-order actions are new subscribers (quality attribute: **modifiability**).
- **Positive:** RabbitMQ is open-source, self-hosted and known by two developers (quality attributes: **cost**, **operability**).
- **Negative:** confirmation emails are delayed during peaks. At 200 orders/s, each minute of peak adds 9,000 emails, which take 3 minutes to drain at 50/s (150 queued/s ÷ 50/s, after the peak ends). Customers may wait minutes for the email (quality attribute: **usability**).
- **Negative:** a new stateful cluster to operate, monitor and upgrade (quality attributes: **cost**, **operability**).
- **Negative:** the flow is eventually consistent and spans several services; harder to trace (quality attributes: **observability**, **testability**). The existing OpenTelemetry instrumentation (`main.go` l.38–42, l.134–140) should propagate trace context into message headers.
- **Risks and mitigations:**
  - *Event lost if RabbitMQ is down or the publish times out*, because checkout has no outbox store (`main.go` l.66–86). Mitigate with quorum queues, background retry, and an alert on publish failures; a durable outbox would need a new order store (Open question).
  - *Duplicate deliveries → double points.* `processed_orders` unique constraint in `loyaltyservice` (see Decision 4).
  - *Duplicate emails* when a MailFast call times out but actually succeeded. Whether MailFast supports idempotency keys is **not evidenced** (Open question).
  - *The 50/s limit is exceeded by scaling the email consumer.* Enforce one shared limit (e.g. a single active consumer, or a shared token bucket), not one per replica.
  - *Event schema changes break consumers.* Version the `OrderPlaced` message (protobuf, like `demo.proto`) and add only optional fields.

## Validation

- **Load test** at 200 orders/s with the Locust `loadgenerator` (as in ADR-0001): `PlaceOrder` p95 ≤ 500 ms.
- **Fault injection:** stop MailFast (or a stub) and `loyaltyservice` during the load test: 0 failed orders; queues drain after recovery; loyalty points awarded ≤ 1 min after `loyaltyservice` recovers for new orders.
- **Duplicate test:** redeliver the same `OrderPlaced` events: points awarded exactly once.
- **Metrics/alerts:** queue depth and age per queue, `loyalty` message age < 1 min, MailFast send rate ≤ 50/s and error rate, dead-letter count, publish failures in checkout.
- **Review** after the 11.11 sale and when the first next-year action is added.

## Open questions

- If RabbitMQ is unavailable, is losing the post-order event acceptable, or must we add an order store / outbox to `checkoutservice`? (The scenario only says the order must not fail when email, loyalty or analytics is down; it says nothing about the broker.)
- How long does the 11.11 peak of 200 orders/s last? This sets the maximum email backlog and how late emails can be.
- What is the acceptable delay for a confirmation email?
- Does MailFast support idempotency keys, batching, or a higher rate limit for sale days?
- Loyalty rules: how many points per order (based on total? currency, since `PlaceOrder` uses `user_currency`)? Which database will `loyaltyservice` use?
- What does marketing need from the analytics event: which fields, which store, how fresh?
- Should the email confirmation be removed from `checkoutservice`'s gRPC dependencies entirely (`EMAIL_SERVICE_ADDR`, `main.go` l.115), or kept as a fallback?
- Who are the deciders?
