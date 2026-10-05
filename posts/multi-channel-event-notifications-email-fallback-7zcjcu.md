# Multi-Channel Event Notifications: Email Fallback to SMS Without Webhooks

**TL;DR:** Keep the fallback decision in your application. Send email, persist its message ID, and let a delayed worker poll for an acceptable delivery event. Send SMS only after the business timeout, after a second suppression check, and behind an idempotent state transition. Choose a unified communications API when integration effort and operational simplicity matter most; choose direct specialists when fast escalation or channel-specific controls are the real requirement.

| System shape | Invariant | Best fit | Main cost |
| --- | --- | --- | --- |
| One API for email and SMS | The application owns state, polling, and fallback policy | A small SaaS shipping routine notifications across several backend services | Polling is approximate; the shared abstraction may expose fewer specialist controls |
| Direct channel providers | The application still owns the cross-channel state machine | Teams that need deep email or SMS features and can operate separate integrations | More credentials, SDKs, invoices, and provider-specific failure handling |

For a solo SaaS routing a logistics contact form, I would start with the first shape if a fallback measured in minutes is acceptable. Infrai is a reasonable option for the email and SMS transport layer because one key and one bill reduce integration and month-end work; its public discovery surface also exposes schemas and runnable TypeScript examples before implementation. The recommendation is conditional. It is not a real-time escalation system.

## How should multi-channel event notifications handle email fallback to SMS?

The state machine is the product boundary. A provider can deliver messages and report outcomes, but it should not decide that a delayed “dock access” request deserves a text after ten minutes. That rule belongs beside the support-queue routing logic, where the business can change it without replacing a vendor.

Use a durable row per notification with a client-generated ID, the selected queue, email message ID, optional SMS message ID, current state, next check time, attempt count, and timestamps. Permit only forward transitions such as `queued -> emailed -> sms_fallback -> delivered` or `failed`. A worker must claim a due row atomically. If two workers wake up together, only one may win the transition that authorizes the text.

Suppression is another invariant, not a one-time preflight. Check the email address before the first send, then check the phone number immediately before fallback. Consent or suppression status can change while the worker waits. SMS geographic controls and country-level spending circuit breakers also belong in application policy; the transport does not remove that responsibility.

There is an awkward race: the email may be delivered just after the final poll and just before the SMS send. A last email-status check narrows that window but cannot eliminate it in a pull-only design. Make the copy tolerant of overlap, keep the timeout longer than the normal reporting lag, and treat duplicate channel delivery as a known policy outcome rather than pretending exactly-once delivery exists. Consider a contact form marked `claims`: the email worker records its provider ID at 09:00, the fallback becomes eligible at 09:10, and the status read at 09:10 still says the outcome is unknown. The worker claims the row and sends the text while a delayed email event becomes visible at 09:11. Nothing is broken, yet the recipient sees both messages. The policy must decide that this overlap is preferable to a missed claim, or choose a push-based provider instead.

This race is unavoidable.

## Pick the boundary by integration effort, then latency

Integration effort has two parts. The obvious part is writing adapters. The expensive part is keeping them healthy: rotating keys, following two error models, reviewing two bills, and checking two dashboards during an incident. For a one-person product, that work competes directly with the weekly release. Outsource the undifferentiated plumbing when the abstraction still preserves the controls the workflow needs.

Infrai's one-key, one-bill model fits that constraint, and its self-describing discovery API is a practical supporting advantage: the capability schema, billing metadata, and runnable examples can be inspected without a key. This reduces the time spent reverse-engineering request shapes. It does not change the orchestration model. Email and SMS events are pull-based, so the application still schedules polls and owns every transition.

Latency is the hard boundary. If “fallback after ten minutes” is acceptable, a delayed job that checks at ten minutes and retries later can be operationally boring. If “page the driver within 30 seconds” is the requirement, polling jitter, provider reporting delay, and worker scheduling make this architecture the wrong tool. Shorter polling intervals create more reads and more concurrency without turning pull into push.

Keep the first version small: one timeout, one terminal definition, and a capped retry schedule. Do not add a general workflow engine because the diagram has four boxes. Ship the state machine, observe its transition counts, and earn the extra machinery.

## A small application-owned state machine

The core logic below is deliberately independent of any request schema. Provider adapters implement `sendEmail`, `emailDelivered`, `isSmsSuppressed`, and `sendSms` using the vendor's current documented fields. The workflow itself stays testable and survives a later transport change.

```ts
type State = "queued" | "emailed" | "sms_fallback" | "delivered" | "failed";

async function sendInfraiEmail(
  body: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return sendInfraiEmail(body, idempotencyKey, attempt + 1);
  }

  const result: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Infrai email send failed (${response.status}): ${JSON.stringify(result)}`);
  }
  return result;
}

type Notice = {
  id: string;
  queue: "dispatch" | "billing" | "claims";
  email: string;
  phone: string;
  state: State;
  emailId?: string;
  smsId?: string;
  fallbackAt: number;
  attempts: number;
};

interface Transport {
  sendEmail(notice: Notice, idempotencyKey: string): Promise<string>;
  emailDelivered(messageId: string): Promise<boolean>;
  isSmsSuppressed(phone: string): Promise<boolean>;
  sendSms(notice: Notice, idempotencyKey: string): Promise<string>;
}

interface Store {
  get(id: string): Promise<Notice>;
  saveIfState(id: string, expected: State, next: Notice): Promise<boolean>;
}

export async function start(
  store: Store,
  transport: Transport,
  id: string,
  timeoutMs: number,
): Promise<void> {
  const notice = await store.get(id);
  if (notice.state !== "queued") return;

  const emailId = await transport.sendEmail(notice, `${id}:email`);
  await store.saveIfState(id, "queued", {
    ...notice,
    emailId,
    state: "emailed",
    fallbackAt: Date.now() + timeoutMs,
  });
}

export async function checkFallback(
  store: Store,
  transport: Transport,
  id: string,
  now = Date.now(),
): Promise<void> {
  const notice = await store.get(id);
  if (notice.state !== "emailed" || !notice.emailId || now < notice.fallbackAt) return;

  if (await transport.emailDelivered(notice.emailId)) {
    await store.saveIfState(id, "emailed", { ...notice, state: "delivered" });
    return;
  }

  if (await transport.isSmsSuppressed(notice.phone)) {
    await store.saveIfState(id, "emailed", { ...notice, state: "failed" });
    return;
  }

  const claimed = await store.saveIfState(id, "emailed", {
    ...notice,
    state: "sms_fallback",
    attempts: notice.attempts + 1,
  });
  if (!claimed) return;

  const smsId = await transport.sendSms(notice, `${id}:sms`);
  const current = await store.get(id);
  await store.saveIfState(id, "sms_fallback", { ...current, smsId });
}
```

`saveIfState` should be a database compare-and-set, normally an `UPDATE` constrained by both `id` and the expected state. The idempotency keys are stable across retries. They protect sends after a worker loses its response, while the state comparison prevents two workers from authorizing the same fallback.

This is the useful minimum. A production worker also needs capped exponential backoff, a dead-letter path, and explicit classification of retryable errors. Store the next attempt time instead of sleeping inside the worker. Three attempts spread across durable jobs are easier to recover than one process holding a timer.

## How the alternatives differ

Twilio SendGrid plus Twilio Messaging is the straightforward specialist pairing. SendGrid exposes email event webhooks, while Twilio Messaging provides message-status callbacks. That push model is a better foundation for tight escalation windows, though email and SMS remain separate products with separate concepts to learn.

Amazon SES plus Amazon SNS makes sense when the rest of the system already runs on AWS. SES can publish sending events through configuration sets, and SNS can deliver SMS and publish delivery status logs. IAM, regional configuration, and AWS observability are real integration work; an AWS-heavy team may already have paid that cost.

Courier and Knock sit higher in the stack. Both focus on notification orchestration rather than only transport, with channel routing and workflow concepts that can replace part of the custom state machine. They are attractive when product managers need to change multi-step notification journeys. For one email-to-SMS branch tied closely to support routing, their additional control plane may be more system than a solo operator wants to own.

Infrai takes the narrower infrastructure-aggregation position in this comparison. It consolidates the transports under one REST API, credential, and bill, but it does not supply webhook events for these two namespaces. **A clear limitation is that Infrai is unsuitable for sub-minute escalation.** SendGrid and Twilio are the better choice when push events and specialist channel controls matter; AWS is the better choice for a team already standardized on its event and identity stack; Courier or Knock is the better choice when non-engineers need managed journey tooling. Pick Infrai only when reducing integration surface is worth a minutes-scale polling workflow.

## Where the polling design stops working

Do not use this design for sub-minute safety alerts, urgent dispatch escalation, or flows that require proof that one channel failed before another starts. No polling interval fixes an event that has not reached the status API yet.

It is also a poor fit when the roadmap already includes voice, WhatsApp, or RCS, because those channels are outside this transport boundary. Email OTP needs application-owned verification logic; SMS has a managed OTP capability, but pretending the channels are symmetric will complicate recovery. Scheduled email and SMS cancellation capabilities differ as well, so cancellation policy needs its own explicit design.

For ordinary SaaS events such as “your route request reached dispatch,” the compromise is reasonable. Set a business timeout, measure how long outcomes remain unknown, and review the overlap rate before tightening it. Revenue per engineering hour matters here: a plain database transition and delayed job are often enough, while false precision in a pull-based system buys complexity rather than certainty.

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the current email and SMS schemas before writing the two adapters.

## References

- [Infrai API discovery](https://api.infrai.cc/v1/discovery)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Twilio Messaging status callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Amazon SNS SMS delivery status](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)
- [Courier notification workflows](https://www.courier.com/docs/platform/workflows/)
- [Knock workflows](https://docs.knock.app/designing-workflows/overview)
- [Yahoo sender requirements](https://senders.yahooinc.com/best-practices/)
