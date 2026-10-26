Agent: Claude Code (VS Code extension), model Claude Opus 5.5 (`claude-opus-5-5`). Run a (zero-shot, context: none).

# ADR-0002: Publish an "order placed" event and handle post-order actions asynchronously

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** Not given (see Open questions)

## Context and problem

Requirement R-12: after an order is placed, Online Boutique must

1. send the order confirmation email through **MailFast**, a new external email provider that replaces the mock `emailservice` logic;
2. award **loyalty points** through a new `loyaltyservice` (to be built);
3. record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

The problem: how should the service that places orders trigger these follow-up actions? The number of actions will grow. One of them depends on an external provider we don't control (MailFast), and one is a new internal service.

Not given in the requirement: how orders are placed today, latency or throughput targets, MailFast's limits or SLA, whether loyalty points must be awarded exactly once, and what infrastructure the team already runs. These are listed under **Open questions**.

## Decision drivers

1. **Modifiability / extensibility:** adding SMS, fraud scoring and a recommendations update next year should not mean changing the order-placement code each time.
2. **Availability of order placement:** an outage at MailFast, `loyaltyservice` or analytics should not stop customers from placing orders.
3. **Reliability of each action:** confirmation emails and loyalty points must not be silently lost, and points should not be awarded twice.
4. **Performance:** post-order work should not add noticeably to the customer-facing order latency (no target given).
5. **Operational cost and complexity** for the team.

## Considered options

1. **Synchronous calls:** the order-placement service calls MailFast (or an email adapter), `loyaltyservice` and analytics directly, one after another, before replying.
2. **Asynchronous calls inside the order service:** the order service replies to the customer, then makes the same calls in the background (goroutine/thread/task queue in process).
3. **Publish an `OrderPlaced` event to a message broker** (publish/subscribe). Email, loyalty and analytics each subscribe as independent consumers; new actions add new subscribers.
4. **Workflow/orchestration engine:** a workflow defines the post-order steps, with retries and state per step.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls | Simplest; no new infrastructure; caller knows straight away if a step failed | Order latency = sum of all calls; a MailFast or loyalty outage fails or slows the order; every new action changes the order service (poor modifiability) |
| 2. In-process async | Customer reply isn't delayed; no new infrastructure | Work is lost if the process crashes after replying; retries must be hand-written; still coupled: every new action changes the order service |
| 3. Event + broker | Order service only publishes one event, so it doesn't change when actions are added; each consumer fails, retries and scales on its own; consumers can absorb bursts and throttle calls to MailFast | New infrastructure to run and monitor; eventual consistency (email and points arrive later); at-least-once delivery means consumers must be idempotent; needs a reliable way to publish the event when the order is saved |
| 4. Workflow engine | Explicit, visible steps; built-in retries, timeouts and compensation | Heaviest option; the order flow becomes coupled to one workflow definition that must change for each new action; new technology for the team |

## Decision

Choose **option 3: publish an `OrderPlaced` event to a message broker** after an order is successfully placed. Separate consumers handle:

- **Email consumer:** calls MailFast to send the confirmation, replacing the mock `emailservice` logic.
- **`loyaltyservice`:** awards points, idempotently keyed on the order ID.
- **Analytics consumer:** records the "order placed" event for marketing.

Reasons: it is the only option where next year's actions (SMS, fraud scoring, recommendations) can be added **without changing the order service**, which is the strongest driver. It also keeps order placement available when MailFast or `loyaltyservice` is down.

The specific broker product is **not chosen here**, because the team's existing infrastructure and skills are not given (see Open questions).

## Consequences

- **Positive:** new post-order actions are new subscribers, with no change to order placement (quality attribute: **modifiability**).
- **Positive:** a MailFast or `loyaltyservice` outage doesn't block orders; events wait in the broker and are retried (quality attribute: **availability**).
- **Positive:** order latency no longer includes the email, loyalty and analytics calls (quality attribute: **performance**).
- **Positive:** consumers scale independently and can absorb order bursts (quality attribute: **scalability**).
- **Negative:** a new broker to deploy, secure, monitor and pay for (quality attributes: **cost**, **operability**).
- **Negative:** email and points are eventually consistent; a customer may see the order before the email or points arrive (quality attribute: **usability / consistency**).
- **Negative:** end-to-end flow is harder to trace across services (quality attribute: **observability / testability**).
- **Risks and mitigations:**
  - *Duplicate delivery → points awarded twice.* Consumers are idempotent: `loyaltyservice` records processed order IDs and ignores repeats.
  - *Event lost if the order is saved but the publish fails (or the reverse).* Use a transactional outbox, or publish only after the order is confirmed and retry. Which one fits depends on how orders are stored today (Open question).
  - *MailFast limits or outages.* The email consumer retries with backoff and sends permanent failures to a dead-letter queue. Throttling depends on MailFast's limits (Open question).
  - *Event schema changes break consumers.* Version the `OrderPlaced` schema and only add optional fields.

## Validation

- A load test that places orders while MailFast and `loyaltyservice` are stopped: orders still succeed, and the backlog drains once they are restarted.
- Fault-injection test: deliver the same event twice and check that points are awarded once.
- Metrics: broker queue depth and consumer lag, the email send failure rate, dead-letter count, order placement latency before and after.
- Review this ADR when the first next-year action (e.g. SMS) is added: it should need no change to order placement.

## Open questions

- How are orders placed today (which service, synchronous or not, is there an order database)? This decides how the event is published reliably.
- Latency and throughput targets for order placement, and expected peak order volume.
- MailFast's rate limit, SLA, cost per email and failure behaviour; does it offer idempotency keys or webhooks?
- Must loyalty points be awarded **exactly once**? Within what time after the order?
- How quickly must the confirmation email arrive?
- Which broker or messaging infrastructure does the team already run or know?
- What does the marketing team need for analytics: real time, or a batch/export is acceptable? Where is the data stored?
- Who are the deciders and owners of `loyaltyservice` and the email integration?
