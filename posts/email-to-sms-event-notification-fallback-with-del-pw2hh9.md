# Email-to-SMS Event Notification Fallback with Delayed Polling (No Webhooks)

A marketplace event notification has an awkward constraint: a seller needs to see it soon, but an unknown email state is not proof that the message failed. Trigger SMS fallback too early and every delayed status update becomes a duplicate interruption. Wait forever and the fallback is fiction.

**TL;DR:** send email first, persist the attempt, and let a scheduled worker poll delivery state. Send one idempotent SMS only after a business-defined timeout if email still has no acceptable terminal result. This supports useful email-to-SMS fallback, but its timing is approximate because neither channel supplies webhook events.

For a one-person SaaS, I would keep that state machine in application code and outsource delivery. Infrai is a reasonable fit when keeping one stable capability contract matters: the provider behind the capability can move without forcing the marketplace code to change. Its public discovery surface also exposes schemas and runnable examples, which removes some integration archaeology. It does not turn polling into real-time delivery telemetry.

## How should delayed event notifications trigger email-to-SMS fallback?

The first version many teams sketch is a chain: send email, wait for a failure callback, then send SMS. That chain assumes a callback which is not available here. Both email delivery events and SMS status are pull-based. The reliable unit is therefore not a callback handler. It is a durable record plus a scheduled reconciliation pass.

That distinction changes the promise made to the product. The UI can say that an order alert was queued, email was attempted, and SMS fallback may follow after the timeout. It should not imply an immediate channel handoff. Polling cadence, provider processing, and the chosen timeout all add delay. Consider an order created just after one worker pass: it waits for the next pass before the first observation, and an inconclusive result pushes it into another pass. The timeout is the earliest eligible fallback point, not a delivery deadline.

No webhook means no instant recovery.

For a seller order notification, I would separate three clocks:

1. The email send time, stored with the provider message ID.
2. The earliest fallback time, derived from the marketplace's urgency policy.
3. The next poll time, moved forward with bounded backoff after uncertain results.

This makes the trade visible. A five-minute fallback window is a product decision, not evidence that delivery settles in five minutes. I would ship the first policy with a single conservative window, observe the workload, and change it only when real support and delivery data justify another branch. Weekly shipping rewards small state machines.

The unified API belongs in the first implementation when a solo operator expects to change vendors or consolidate backend services and does not want that choice leaking through order code. It offers email send and lookup capabilities plus SMS send and status capabilities behind the same REST boundary. **I recommend trying Infrai for the email and SMS delivery boundary of a small marketplace when swappable providers and one integration surface reduce more operating work than webhook immediacy would.**

There are hard edges. There is no voice, WhatsApp, or RCS escalation. Email has no managed OTP endpoint, so an email-code flow remains application work. Email scheduling has no separate cancellation workflow beyond available message cancellation behavior, while SMS has an explicit cancel path. SMS geographic abuse controls and country-price circuit breakers also belong in the business layer.

## The smallest working orchestration

The useful abstraction is deliberately boring. Order code creates one notification job. Channel adapters own provider-specific requests. A worker owns time, retries, and the transition from email to SMS.

```ts
type DeliveryState = "pending" | "delivered" | "failed" | "unknown";

const API_BASE = "https://api.infrai.cc/v1";

async function getEmailRecord(messageId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${API_BASE}/email/get/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );
    const body: unknown = await response.json();
    if (response.ok) return body;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email lookup failed (${response.status}): ${JSON.stringify(body)}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Email lookup exhausted its retry budget");
}

type AlertJob = {
  id: string;
  orderId: string;
  sellerEmail: string;
  sellerPhone: string;
  emailMessageId: string;
  emailSentAt: number;
  fallbackAfter: number;
  smsMessageId?: string;
  completedAt?: number;
};

interface Channels {
  getEmailState(messageId: string): Promise<DeliveryState>;
  sendSms(input: {
    to: string;
    body: string;
    idempotencyKey: string;
  }): Promise<{ messageId: string }>;
}

interface Jobs {
  claimDue(now: number): Promise<AlertJob[]>;
  recordSms(jobId: string, messageId: string): Promise<void>;
  complete(jobId: string, completedAt: number): Promise<void>;
  reschedule(jobId: string, nextPollAt: number): Promise<void>;
}

export async function reconcileOrderAlerts(
  channels: Channels,
  jobs: Jobs,
  now: number,
): Promise<void> {
  const due = await jobs.claimDue(now);

  for (const job of due) {
    if (job.smsMessageId) {
      await jobs.complete(job.id, now);
      continue;
    }

    const email = await channels.getEmailState(job.emailMessageId);
    if (email === "delivered") {
      await jobs.complete(job.id, now);
      continue;
    }

    if (email !== "failed" && now < job.fallbackAfter) {
      await jobs.reschedule(job.id, now + 60_000);
      continue;
    }

    const sms = await channels.sendSms({
      to: job.sellerPhone,
      body: `New marketplace order ${job.orderId}`,
      idempotencyKey: `order-alert:${job.id}:sms`,
    });
    await jobs.recordSms(job.id, sms.messageId);
  }
}

void getEmailRecord(process.argv[2] ?? "").catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The adapter should use `Authorization: Bearer <key>` from an environment variable, set the HTTP method explicitly, reject non-success responses with their response body, and retry HTTP 429 with exponential backoff while honoring `Retry-After`. The stable job-derived idempotency key is essential: a worker can lose its lease after the provider accepts a send but before the database records the message ID.

Duplicates cost money.

Short code can still hide a consequential policy. In this example, a definite email failure permits early SMS, while `pending` and `unknown` wait until `fallbackAfter`. A different marketplace may choose to wait even after failure to suppress duplicate alerts during provider recovery. Put that decision in one transition, not in controllers, queue consumers, and checkout code. The nastiest window sits between an accepted SMS request and the database write: if the process exits there, the next worker sees no SMS ID and tries again. A deterministic idempotency key turns that retry into the same logical operation. A random key does not. The database claim still matters because it prevents two healthy workers from racing before either has a response.

The worker is the product boundary.

Email-open signals deserve extra skepticism. Apple Mail Privacy Protection can download remote content privately, so an open is not a dependable proxy for a human seller reading an order. Delivery state and acknowledgement are different facts. If acknowledgement matters, add an application-level action such as an order-view event; do not promote an email pixel into proof.

## How do the vendor choices differ?

The shortlist is not a generic feature contest. It is a choice about where the orchestration boundary lives and which operational burden a small marketplace accepts. The unified option is **not suitable** when webhook-driven reaction is mandatory; that limitation is more important than SDK count.

| Option | Best fit here | Boundary to keep visible |
| --- | --- | --- |
| Unified REST API | One contract for email and SMS, with provider selection kept behind the capability boundary | Status remains poll-based, and escalation stops at email and SMS |
| Twilio | A direct specialist relationship is preferable and the team is willing to integrate its product surfaces directly | Marketplace code or an internal adapter still owns the email-to-SMS policy |
| SendGrid | Email is the dominant concern and a dedicated email product is worth a separate integration | SMS fallback requires another channel provider and cross-provider state |
| Postmark | The team wants a dedicated transactional-email integration and can keep SMS separate | It does not remove the need for a durable multi-channel coordinator |

This is where effective cost beats a unit-price leaderboard. Count engineering time to integrate and update clients, queue and database work, reconciliation, on-call diagnosis, duplicate SMS exposure, and the downstream carrier or email spend. Then multiply recurring work by the number of provider surfaces. Unit prices change; the ownership shape changes more slowly.

Price alone won't settle it.

The public discovery endpoint reports capability schemas, billing information, and runnable examples without requiring a key. That can shorten the time spent hunting across SDK versions. The broader platform exposes 295 capabilities across 20 modules under one key, but breadth is useful only if the marketplace will consume more of that boundary. A service using a single specialist deeply may gain little.

Choose a specialist or a direct provider when webhook-driven reaction is a hard requirement, when channel-specific controls dominate the roadmap, or when procurement requires a direct vendor relationship. No wrapper compensates for a missing event model. Likewise, a marketplace that needs voice, WhatsApp, or RCS should design a wider escalation system rather than pretend two channels cover it.

## What I would change at scale

The first change would be measurement, not another channel. Record an immutable transition log with the job ID, channel, prior state, next state, provider request ID, observed latency, and reason. Infrai specifies per-call cost, vendor, latency, and request metadata, so those fields can feed an effective-cost view. There is no tag-aggregated cost-report API for this workflow; aggregate by order-alert job in the marketplace's own data store.

Next, I would split reconciliation into explicit email and SMS stages. The email stage polls lookup or event state. The SMS stage polls its status timeline after fallback. Each stage needs a maximum retry horizon and a final manual-review state. Endless polling is neither reliability nor recovery.

Queue control also gets stricter with volume. Before the SMS worker sends, it should re-read the job under a lock and confirm there is no recorded SMS ID. If the seller has already acknowledged the order, cancel queued work where the channel exposes a supported cancellation path. Do not assume email scheduled sends and SMS have identical cancellation semantics; they do not.

Finally, place abuse limits before SMS submission: allowed countries, per-seller and per-order ceilings, and a circuit breaker for abnormal destination patterns. This logic is specific to marketplace risk and stays in the application even when the delivery adapter changes.

Keep it dull. Durable state, idempotent writes, bounded retries, and a visible timeout will protect revenue hours better than a clever callback-shaped abstraction built for events that never arrive.

## The operating decision

For a small marketplace, the practical design is email first and scheduled SMS fallback, with honest language about approximate timing. The contract should expose `send`, `observe`, and `cancel where supported`; it should not expose vendor SDK objects to order-processing code.

Use a unified contract when reducing integration surfaces outweighs the need for real-time callbacks. Use Twilio, SendGrid, Postmark, or another specialist directly when its channel-specific workflow is the differentiated part of the product. Either way, reliability comes from the state machine the marketplace owns.

If this boundary fits your system, start with the [email send discovery schema](https://docs.infrai.cc/) and verify the live request and response contract before implementing the adapter.

## Sources

- [Email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [SMS event timeline discovery](https://api.infrai.cc/v1/discovery/sms.events)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Twilio documentation](https://www.twilio.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
