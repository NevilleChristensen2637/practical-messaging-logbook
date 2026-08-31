# Node.js Send-Later Logistics: User Reminder Retries Across Queue Limits

Decision rule: a Node.js delayed queue message may send a user reminder later or retry it only when the system preserves the original delivery identity and records the transition. The timer decides *when* work becomes eligible. Replay policy decides whether the same logistics notification may run again.

| Governance choice | Replay authority | Audit evidence | Use it when |
|---|---|---|---|
| Durable delivery record plus delayed message | Guarded state transition | Identity, actor, time, and outcome | A reminder can cross seven days or trigger an outbound webhook |
| Queue message as the complete record | Broker redelivery or operator action | Queue history and downstream logs | Work is short-lived and fully rebuildable |
| Due-time database scan | Conditional row claim | Delivery-row history | Timing can be approximate and volume is modest |

**Recommendation:** use the durable delivery record when a shipment reminder crosses the seven-day queue horizon or calls an external webhook. Make automated retry and manual dead-letter replay use the same guarded transition, with the same identity. This choice is driven by recovery governance: an operator should be able to tell who replayed what, why it was eligible, and whether the destination had already accepted it.

This is also a revenue-per-hour choice. A one-person SaaS should outsource the undifferentiated wake-up mechanism and keep the business invariant in code it can test. Ship weekly. Don't spend a release teaching a timer what a shipment means.

## What replay policy should a Node.js delayed queue use for user reminder retries?

Name the invariant before choosing queue settings: one shipment event, one `deliveryId`, one externally visible effect. A useful identity might combine the shipment booking, reminder kind, and schedule revision. It must be created when the reminder is created, not inside the worker. If a renewed booking creates a materially different reminder, increment the revision deliberately; a timeout does not earn a new identity.

That distinction matters at the worst boundary in the system. Imagine the destination accepts a webhook, but the sender stops before recording success. The queue will eventually make the work eligible again. The second worker cannot know from local state whether the first request took effect, so it sends the same `deliveryId`. The receiving side stores that key with the accepted result and returns the prior outcome for a repeat. Retries then repeat transport, not business intent.

Keep the message lean: `deliveryId` and `availableAt` are enough for the queue adapter. The worker loads the current shipment record before sending. A canceled booking can be suppressed, a revised booking can point to a different delivery identity, and personal or mutable destination data does not have to sit in a delayed payload for days.

Seven days is a design horizon here, not a universal queue guarantee. A scheduler promotes only records due within that horizon; records farther out remain in durable storage. The exact margin should come from the queue's documented delay and retention behavior plus a restore drill. I'm not sure a fixed safety margin belongs in a reusable library because deployment pauses, clock discipline, and recovery objectives vary by system.

Urgency is separate from due time. A priority queue changes which eligible item gets attention first; it does not turn a far-future reminder into a durable calendar. RabbitMQ documents priority queues as a distinct queue feature. Treating priority as scheduling tends to hide the business deadline inside broker configuration, precisely where a small team has the least useful context during an incident.

One more rule: acknowledge the queue message only after the state transition commits. If the process stops earlier, the lease can expire and the message can be presented again under the same identity. If it acknowledges first, the reminder can disappear between transport success and durable state.

The ambiguity stays.

What is controllable is the effect inside that window. Sign the exact body together with the delivery identity and timestamp. RFC 2104 defines HMAC as keyed hashing for message authentication; the acceptance window and idempotency retention are application policy. The receiver needs to retain accepted identities for at least as long as the sender can retry or replay them.

## Write the replay policy before the worker

A timer gives an item eligibility. The delivery row gives it meaning. Use explicit states such as `scheduled`, `leased`, `retry_wait`, `delivered`, `suppressed`, and `dead_letter`, with conditional updates guarded by a lease token. Two workers may race to claim the same row, but only one conditional update should acquire the active lease.

Retries need classification before delay calculation. A timeout or HTTP `408` leaves the outcome uncertain, while HTTP `429` asks the sender to slow down. Both can return to `retry_wait`; a contractually permanent rejection should go directly to `dead_letter`. The backoff policy can use capped exponential delay with jitter, but it must not manufacture a new delivery identity. Short and sharp: retry the attempt, not the event.

Dead letters are work, not storage. Keep the delivery identity, attempt count, last classified outcome, next eligible time, and safe diagnostic metadata. Do not copy signing secrets or unnecessary customer data. A replay is a guarded transition from `dead_letter` back to `retry_wait`, preserving the original identity and recording who initiated it.

This makes operations boring — a compliment here. An operator can answer whether a shipment reminder is waiting, leased, suppressed, delivered, or exhausted without inferring business state from queue depth. Queue depth still matters, but age of the oldest eligible delivery is the stronger alert: a small queue can contain one very late, high-value reminder. The replay command should require an expected current state, preserve `deliveryId`, record an actor and reason, and refuse to create a second active lease. Those controls cost more than a “requeue” button, yet they turn recovery into a reviewable business action instead of an invisible mutation of broker state. Your mileage may vary on how much approval a low-risk reminder needs; the irreversible effect at the destination should set that bar.

The catch is receiver cooperation. Sender-side idempotency prevents concurrent local claims and accidental new identities, but it cannot guarantee one external effect if the webhook receiver ignores the key. For destinations without a deduplication contract, be honest about the limit: retrying uncertain outcomes can duplicate delivery, while refusing to retry can lose delivery. That policy belongs in the integration contract, not in a generic queue helper.

## Enforce the policy in TypeScript

This example keeps storage and queue mechanics behind interfaces. The handler uses a stable `deliveryId`, signs the exact serialized body, and returns an explicit transition for the repository to commit before the adapter acknowledges the message. All times are injected so the seven-day boundary and retry schedule can be tested in milliseconds.

```ts
import { createHmac } from "node:crypto";

type Delivery = {
  deliveryId: string;
  leaseToken: string;
  shipmentId: string;
  destination: string;
  state: "leased";
  attempt: number;
  dueAt: Date;
  shipmentStatus: "active" | "renewed" | "canceled";
};

type DeliveryResult =
  | { kind: "accepted" }
  | { kind: "timeout" }
  | { kind: "response"; status: number; retryAfterSeconds?: number };

type Transition =
  | { state: "delivered"; at: Date }
  | { state: "suppressed"; reason: string }
  | { state: "retry_wait"; availableAt: Date; outcome: string }
  | { state: "dead_letter"; outcome: string };

type Sender = (request: {
  destination: string;
  headers: Record<string, string>;
  body: string;
}) => Promise<DeliveryResult>;

const maxAttempts = 8;

function backoffMs(attempt: number, random: () => number): number {
  const capMs = 6 * 60 * 60 * 1_000;
  const ceilingMs = Math.min(capMs, 30_000 * 2 ** attempt);
  return Math.floor(ceilingMs / 2 + random() * ceilingMs / 2);
}

function sign(
  secret: string,
  timestamp: string,
  deliveryId: string,
  body: string,
): string {
  return createHmac("sha256", secret)
    .update(`${timestamp}.${deliveryId}.${body}`)
    .digest("hex");
}

export async function attemptDelivery(
  delivery: Delivery,
  secret: string,
  send: Sender,
  now: Date,
  random: () => number,
): Promise<Transition> {
  if (delivery.shipmentStatus !== "active") {
    return {
      state: "suppressed",
      reason: `shipment-${delivery.shipmentStatus}`,
    };
  }

  const timestamp = now.toISOString();
  const body = JSON.stringify({
    type: "shipment.expiry-reminder",
    deliveryId: delivery.deliveryId,
    shipmentId: delivery.shipmentId,
  });
  const result = await send({
    destination: delivery.destination,
    headers: {
      "content-type": "application/json",
      "idempotency-key": delivery.deliveryId,
      "x-delivery-timestamp": timestamp,
      "x-delivery-signature": sign(
        secret,
        timestamp,
        delivery.deliveryId,
        body,
      ),
    },
    body,
  });

  if (result.kind === "accepted") {
    return { state: "delivered", at: now };
  }

  const outcome = result.kind === "timeout"
    ? "timeout"
    : `http-${result.status}`;
  const nextAttempt = delivery.attempt + 1;
  const retryable = result.kind === "timeout"
    || (result.kind === "response"
      && (result.status === 408 || result.status === 429));

  if (!retryable || nextAttempt >= maxAttempts) {
    return { state: "dead_letter", outcome };
  }

  const requestedDelayMs = result.kind === "response"
    && result.status === 429
    && result.retryAfterSeconds !== undefined
      ? Math.max(0, result.retryAfterSeconds * 1_000)
      : backoffMs(delivery.attempt, random);

  return {
    state: "retry_wait",
    availableAt: new Date(now.getTime() + requestedDelayMs),
    outcome,
  };
}
```

The repository should commit this transition with a condition on both `deliveryId` and `leaseToken`. The queue adapter acknowledges only after that commit succeeds. On a stale lease, it does neither; the current owner decides the record's next state.

## Can the recovery drill prove the send-later contract?

Testing should follow the failure boundary, not just the happy path. Freeze time one millisecond before the seven-day promotion boundary, then cross it and assert exactly one claim. Return `429` with a retry delay and prove the delivery stays unavailable until the calculated instant. Next, model an accepted request whose local transition is never committed: reacquire the lease, send again, and verify that the second request has byte-for-byte the same body and identity. Exhaust eight attempts, move the item to dead letter, authorize a replay with an actor and reason, and check that replay keeps the identity while adding an audit event. Finally cancel the shipment before its due time and confirm suppression occurs without a send. This is the release gate I would automate for weekly shipping because it exercises policy at the uncertain network boundary, where a locally green worker can still create a duplicate external effect.

One drill. Many claims.

It covers the promotion clock, conditional lease, signer, receiver contract, retry classification, dead-letter transition, and replay authority. A tidy unit test for `backoffMs` doesn't. In staging, also inspect the audit record rather than trusting a success flag: it should link the original schedule, every attempt, the terminal outcome, and the operator-authorized replay without exposing secrets.

## Choose the simpler boundary when governance is cheap

Use a queue-only record when reminders remain comfortably inside documented queue limits, all pending work can be reconstructed, and duplicate effects are harmless or rejected downstream. It removes the promotion scan and delivery table from the weekly shipping path. For a low-stakes internal nudge, that may be the better revenue-per-hour trade.

Use a due-time database scan without delayed messages when volume is modest and minute-level timing is acceptable. An indexed due-time query, bounded claims, and leases can be easier to operate than two moving parts. The cost is repeated polling and coarser timing.

Neither runner-up is suitable for a logistics webhook whose receiver performs a costly, irreversible action without idempotency support. In that case, no sender architecture can promise exactly one external effect across an uncertain network result. Change the receiver contract, introduce a reconciliation workflow, or require human review before replay. More retries are not a substitute.

For a solo operator, the durable-identity design also has a real cost: another table, a promotion job, lease cleanup, replay authorization, and age-based monitoring. Stick with the simpler option when its failure mode is cheap and reconstructable. Add the state machine when a duplicate shipment notification costs more trust than the extra machinery costs engineering time.

## References

- RFC 2104, “HMAC: Keyed-Hashing for Message Authentication”: https://www.rfc-editor.org/rfc/rfc2104
- RabbitMQ, “Priority Queue Support”: https://www.rabbitmq.com/docs/priority
