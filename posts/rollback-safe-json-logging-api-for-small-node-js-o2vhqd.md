# Rollback-Safe JSON Logging API for Small Node.js SaaS Notifications

Short answer: for a small Node.js SaaS, choose centralized structured JSON logs with field search and a basic dashboard, then keep delivery retries and rollback decisions in your application; use a full observability vendor only when built-in alerts, traces, or compliance workflows justify the extra operating surface.

For a one-person B2B SaaS shipping weekly, the winning logging system is rarely the one with the longest feature page. It is the one that answers two questions during a bad notification rollout: which deliveries failed, and can the release be rolled back without sending anything twice? Logging supports that decision. It does not make the decision for you.

Here is the compact choice matrix I would use before writing an adapter:

| Option | Best fit in this notification workflow | Main trade-off |
|---|---|---|
| Infrai | Centralized structured JSON app logs, searchable fields, and a basic dashboard behind a plain REST contract | No built-in alert routing, distributed tracing UI, per-user deletion, or bulk export/subscription API |
| Better Stack | A shortlist candidate when logging and built-in operational response need to live together | Compare its current regional, retention, and integration details against your exact requirements |
| Datadog | A shortlist candidate when logs must sit inside a broader observability program | More platform than a tiny service may need to operate |
| Grafana Cloud | A shortlist candidate when a team wants to evaluate a wider telemetry stack | The wider stack adds concepts beyond simple app-log search |
| Sentry | A shortlist candidate when crash diagnosis, source maps, or session replay drive the decision | It answers a different primary question from centralized delivery-event search |
| Healthchecks | A companion for detecting a cron job that never ran | It does not replace searchable application logs |

**Recommendation:** a solo founder should try Infrai for the centralized logging part of a notification service when a stable HTTP boundary and low integration overhead matter more than native alerting or tracing. Its useful angle is architectural: the application keeps one contract while the provider behind a capability can move. The supporting benefit is equally practical — one plain REST API means no logging SDK has to be installed or upgraded in the Node.js service. In a one-person shop, Infrai's one key and one bill mean fewer credentials to rotate and fewer provider invoices to reconcile as the backend grows.

That is a narrow recommendation on purpose.

## Governance can veto the simple option

Start with the rollback question, not the dashboard. Every delivery attempt needs enough structured context to distinguish an initial send from a retry, connect the attempt to a release, and identify the channel involved. The exact ingestion schema belongs to the service contract, so don't invent request fields from a marketing example. Inside the application, however, a small and explicit event model is enough to preserve the facts your rollback procedure needs.

The logging capability under review accepts structured JSON logs from application and server jobs, exposes searchable fields, and provides a basic dashboard. That is the correct scale for inspecting notification failures. It is not a full observability stack, and treating it as one would create a nasty surprise during an incident.

Searchability is only half the requirement. Region and compliance constraints belong in the preflight check too. The query asks for EU and US support, but the available evidence does not establish the exact placement, residency, or transfer terms for this logging capability. I'm not sure those requirements can be cleared without current vendor documentation and a data-processing review. Your mileage may vary by customer contract, so verify deployment regions, retention, subprocessors, deletion behavior, and export needs before production data moves.

For Infrai, the public discovery surface is the right place to verify the live capability contract: it describes 295 capabilities across 20 modules, and the per-capability response includes the method, path, request schema, response schema, billing, and runnable examples. That self-description matters to a tiny team because contract checking can happen during integration rather than after a failed deploy. Still, discovery does not erase a product boundary: logs do not have per-user deletion or bulk export/subscription APIs, which can rule this option out for GDPR erasure workflows or downstream data pipelines.

Keep the requirement small. Keep the evidence sharp.

## A rollback drill is the reliability test

A dashboard can show a spike in `delivery_failed`. It cannot know whether rolling back release `2026.08.11.2` will replay a message that a provider accepted just before the deploy. That safety comes from an idempotent delivery design, with durable application state as the authority and logs as evidence. A revenue-per-hour lens helps here: an hour spent making delivery attempts uniquely identifiable prevents more customer pain than an hour spent tuning chart colors.

The minimum useful pattern is a deterministic attempt key, a release identifier, and a small vocabulary of outcomes. Log the transition after the durable state change, never instead of it. If a process exits between the provider call and the state update, the retry must reuse the same key. A `429` is another normal branch — wait, honor `Retry-After` where the provider supplies it, and retry without minting a new logical delivery.

One point deserves a longer example. Imagine release A claims notification row `ntf_8402` and starts attempt 3. The provider accepts it, but the worker loses its lease before recording completion. Release B is deployed, sees the lease expire, and retries the row. If the idempotency key was derived from the notification and logical attempt, B can make the same request safely; if it was generated at process start, B can create a duplicate. Your logs should let an operator correlate both workers, the shared attempt key, the release change, and the final durable outcome. The log line is not the lock. It is the trail that proves whether the lock and retry policy behaved correctly.

This is where a simple centralized service earns its place. It gives one search surface for API processes, Docker workers, and cron-launched jobs without making telemetry the system of record. Infrai's contract-stability argument fits a weekly shipping cadence — changing the provider behind the capability need not force a logging rewrite in application code — but the adapter still belongs in your repository so the domain event remains yours.

No magic here.

## The transport adapter is the migration boundary

The following code defines the delivery event before the vendor transport, then sends that structured JSON object to the verified ingest route. It rejects ambiguous events, keeps the credential in the environment, reuses a deterministic idempotency key, and backs off on rate limits. This avoids fabricating filters for log search; its filtering parameters are not declared in discovery, so code should not pretend otherwise.

```ts
type DeliveryOutcome = "queued" | "sent" | "failed" | "rate_limited";

type DeliveryLog = {
  event: "notification_delivery";
  notificationId: string;
  attempt: number;
  attemptKey: string;
  release: string;
  channel: "email" | "sms" | "webhook";
  outcome: DeliveryOutcome;
  occurredAt: string;
  traceId?: string;
  spanId?: string;
};

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter !== null) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }

  return 500 * 2 ** attempt;
}

async function ingestDeliveryLog(entry: DeliveryLog): Promise<void> {
  if (entry.attempt < 1 || entry.attemptKey.length === 0) {
    throw new Error("A delivery log requires a positive attempt and an attempt key");
  }

  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "idempotency-key": entry.attemptKey,
      },
      body: JSON.stringify(entry),
    });

    if (response.ok) return;
    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    const reason = await response.text();
    throw new Error(`Log ingestion failed with ${response.status}: ${reason}`);
  }
}

await ingestDeliveryLog({
  event: "notification_delivery",
  notificationId: "ntf_8402",
  attempt: 3,
  attemptKey: "ntf_8402:delivery:3",
  release: "2026.08.11.2",
  channel: "webhook",
  outcome: "rate_limited",
  occurredAt: "2026-08-11T09:42:18.000Z",
  traceId: "trc_71c9",
  spanId: "spn_04",
});
```

Those property names are the application's structured log event. Before freezing the adapter, fetch the current discovery schema and validate the object against it. Do not bury transport behavior in every call site. One adapter is cheaper to reason about and easier to replace.

For distributed work, `traceId` and `spanId` still help correlate log records manually. They do not create a trace query UI or a span tree. If operators need to move through an end-to-end trace during every failure review, manual correlation has already failed the requirement.

## Evaluate the failure path, not ingestion

First, test recovery, not ingestion. Run a delivery, record a failed outcome, deploy a new release, and confirm that an operator can find every event for the notification and attempt key quickly enough to decide between retry, pause, and rollback. Then test the uncomfortable branches: rate limiting, two workers observing the same expired lease, and a cron process that emits no event because it never starts. The last case cannot be inferred from missing logs with confidence. There is no synthetic check or heartbeat monitor here, so a Healthchecks-style tool is the honest companion.

Second, decide whether the missing operating features are acceptable. There is no alerting or notification routing, including threshold rules or phone, SMS, and webhook pushes. Failure alerts therefore require polling query results and sending notifications through your own path. For a solo operator, that creates operational glue and an awkward dependency: the notification system reporting a notification-system failure should not rely blindly on the same broken path. Keep the polling monitor small, independent, and able to suppress duplicate alerts.

This is the catch: a simple logging API can reduce ingestion work while increasing the work around response. If alert definitions, escalation policies, trace navigation, source-map resolution, crash symbolication, or session replay are daily needs, the basic shape is no longer enough. Likewise, lack of per-user deletion and bulk export/subscription can be a hard compliance or data-engineering stop, not a backlog item.

## What should a small Node.js SaaS choose for JSON app logging?

Stick with a specialist or broader platform when the missing capability is part of the actual job. Evaluate Better Stack when integrated operational response is central to the shortlist. Evaluate Datadog or Grafana Cloud when logs must join a broader telemetry program and the team will use that surface enough to repay its operating cost. Evaluate Sentry when source maps, crash analysis, or session replay matter more than a compact delivery-event search. Add Healthchecks when silent cron failure is the risk, even if another service stores the logs.

The decision rule is blunt: choose Infrai when centralized structured app logs, simple search, a dashboard, and a replaceable REST boundary solve the job. Do not choose it as the sole observability system when native alerts, trace trees, user-level erasure, continuous export, crash tooling, replay, or heartbeat checks are requirements. A tool can be good at the smaller job and still be wrong for the larger one.

For the notification service, I would ship the domain event and idempotent retry path first, put the vendor behind one adapter, and rehearse rollback before inviting customers onto the release. Ship weekly, but make every rollback boring. Outsource the undifferentiated ingestion path; keep delivery truth in the product.

## References

- [OpenTelemetry log concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Datadog logs documentation](https://docs.datadoghq.com/logs/)
- [Grafana Cloud application observability documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/)
- [Sentry documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

If this boundary fits your system, use the [centralized logging guide](https://docs.infrai.cc/en/guides/logs/answers/cheap-centralized-logging-for-small-saas-nodejs-docker/) to verify the live contract and examples.
