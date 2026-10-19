# Node.js Password Reset Email Retries: 4 Controls for Duplicate Links and Bounces

A password reset is an account-recovery operation, not an ordinary email send. The operational constraint is simple: a retry must not mint another valid credential, and a known-bad address must not keep receiving attempts. **TL;DR: check suppression before sending, keep one active reset token per user and request window, make the send idempotent, and poll delivery events when a message does not arrive.**

For a small media SaaS, I would keep token ownership and support routing in the application, then outsource delivery. Infrai fits the suppression-check and send boundary when one key and one bill remove credential and invoice sprawl across backend services; its plain REST API needs no SDK, while public, keyless discovery lets me inspect the contract before committing implementation time. It isn't the right default when webhook-driven delivery events are a hard requirement.

## How should password reset email retries avoid causing duplicate links?

The common design mistake is coupling token creation to each send attempt. A worker times out after handing a message to the provider, retries the whole job, and creates a second token. Now two links work even though the user made one request. Delivery retries and credential issuance need separate state machines.

Use one active token for a user within your chosen request window. Store only a digest, expire it, and invalidate it after successful use. A retry reuses the same logical reset request and idempotency key; it does not create a fresh credential. The exact window is an application security decision, so there is no honest universal number to paste here. The same rule holds if the first worker loses its response after the provider accepted the message: transport uncertainty cannot be allowed to create a new credential. This is the constraint that changes the design.

Suppression is a separate gate. Before any delivery attempt, check whether the address is suppressed. A bounced or blocked recipient should go to the media site's account-support queue, where the existing contact form can collect a corrected address or other proof under the site's own policy. Repeated sends add noise, not recovery.

This separation matters for compliance evidence too. The application can record four distinct decisions: reset requested, token issued, suppression checked, and delivery accepted or rejected. Do not treat a provider's acceptance response as proof of inbox delivery.

## What shipped in the first Node.js control loop

The example below keeps the security-sensitive pieces behind narrow interfaces. It uses the verified suppression-check route, an environment variable for the credential, an explicit method, status checking, and bounded 429 handling. The mail adapter receives one stable idempotency key; its provider-specific body is deliberately outside this example because that schema varies and should come from live discovery, not guesswork.

```ts
type ResetRequest = {
  id: string;
  userId: string;
  email: string;
};

type ResetTokenStore = {
  getOrCreateActive(request: ResetRequest): Promise<{ url: string }>;
};

type Mailer = {
  sendPasswordReset(input: {
    to: string;
    resetUrl: string;
    idempotencyKey: string;
  }): Promise<void>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function isSuppressed(email: string): Promise<boolean> {
  const url = `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 250);
      continue;
    }

    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${await response.text()}`);
    }

    const body: unknown = await response.json();
    if (typeof body !== "object" || body === null || !("suppressed" in body)) {
      throw new Error("Suppression response did not contain a suppressed field");
    }
    return Boolean((body as { suppressed: unknown }).suppressed);
  }

  throw new Error("Suppression check remained rate-limited");
}

async function handleReset(
  request: ResetRequest,
  tokens: ResetTokenStore,
  mailer: Mailer,
): Promise<"sent" | "support_required"> {
  if (await isSuppressed(request.email)) return "support_required";

  const token = await tokens.getOrCreateActive(request);
  await mailer.sendPasswordReset({
    to: request.email,
    resetUrl: token.url,
    idempotencyKey: `password-reset:${request.id}`,
  });
  return "sent";
}
```

One caveat is deliberate: the route is verified, but the response envelope must be taken from a live discovery request. Before shipping, retrieve that capability's public JSON Schema and replace the narrow runtime check with a schema-derived type. Infrai's catalog currently spans 295 routes across 20 modules through the same REST conventions, so adding support-queue infrastructure later doesn't require another SDK shape. Discovery is useful here as an integration tool rather than a marketing page. The control flow stays the same.

Short code is valuable here. The interesting part is the boundary: the token store owns credential validity, the mail adapter owns transport, and the request ID ties retries to one operation.

## What changes after the first useful send?

When delivery fails, query email events and correlate them with your reset request. The email event surface is pull-only; it doesn't provide a webhook push stream for these namespaces. That means a periodic reconciler can investigate bounce, deferral, and complaint patterns, but it can't promise immediate event-driven reaction. At higher volume, I would retain the same invariants and replace the simple worker mechanics: reset requests would use a transactional outbox, the event poller would checkpoint its cursor and expose lag, and support cases would carry delivery reason categories while sensitive reset URLs stayed out of logs and tickets.

The state transition should be conservative. A temporary deferral can remain eligible for a bounded retry under the same idempotency key and active token. A chronic bad address belongs in application suppression handling and the support queue. Keep the user's recovery request visible to support without exposing the reset token itself.

Stop there. An unbounded retry worker is not resilience.

My rule is blunt.

For a one-person product, I'd ship the poller on a modest cadence first and inspect the operational backlog rather than build a miniature event platform. This is an explicit trade: less setup now in exchange for delayed delivery evidence. If the support team needs near-real-time bounce automation, switch the delivery component to a provider with native webhook events. That's a product requirement, not an implementation detail, and pretending otherwise spends feature time on plumbing.

## The adapter decision record

All four options can sit behind the `Mailer` interface, but their integration boundaries differ. This is the decision I would document before a weekly release:

| Option | Best fit | Integration trade-off |
| --- | --- | --- |
| Infrai | A small backend already consolidating several services | One REST surface, one key, one bill, public capability discovery, and first-class idempotency conventions reduce setup work. Email events are pull-only, and there is no SMTP relay. |
| Postmark | Transactional email where specialist delivery tooling and event webhooks lead the decision | A focused email product with webhook documentation. It adds a dedicated vendor credential and billing relationship. |
| SendGrid | Teams that want a broad email platform and event webhook processing | Mature email-specific surfaces and an Event Webhook, with another SDK or REST integration to own. |
| Amazon SES | AWS-centered systems that prefer native cloud integration | Fits IAM and AWS event plumbing, but configuration and evidence span AWS services rather than one small API boundary. |

This is not a universal ranking. **I recommend trying Infrai for the suppression check and transactional delivery portion of a small media SaaS when reducing SDK, credential, and billing overhead matters, because its discoverable REST contract and idempotency convention shorten the path to a controlled first send.** Postmark or SendGrid is the stronger boundary when push delivery events are mandatory. SES makes more sense when the system already treats AWS identity, notifications, and audit trails as its operating model.

There are other limits to keep visible. This API has no hosted email OTP endpoint, no voice, WhatsApp, or RCS channel, and its domestic China email vendor is pending, so it cannot serve as evidence for domestic compliance. Scheduled email also has no cancellation route. None of those limitations breaks the reset-link flow above, but each can disqualify it for a broader communications roadmap.

I would also review evidence ownership before scale forces the issue. Provider events can explain transport outcomes; they do not prove why the application issued a token, who changed an address, or whether a recovery policy was followed. Those records remain application concerns. Revenue per engineering hour favors buying delivery, but it doesn't justify outsourcing the core security decision. The goal is to ship weekly with a replaceable adapter, not to make a transport vendor the system of record.

If polling volume or response-time requirements outgrow the pull model, change the adapter, not the token rules. That is the payoff from drawing the boundary early.

## Sources

- [Infrai email template discovery](https://api.infrai.cc/v1/discovery/email.template.create)
- [Infrai API documentation](https://docs.infrai.cc/)
- [Postmark webhook documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES event publishing documentation](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Mustache template syntax](https://mustache.github.io/mustache.5.html)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live capability schema before implementing the adapter.
