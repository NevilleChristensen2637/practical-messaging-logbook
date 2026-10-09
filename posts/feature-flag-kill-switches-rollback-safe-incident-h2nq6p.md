# Feature Flag Kill Switches — Rollback-Safe Incident Response for Media Systems

A media incident creates two urgent jobs: stop the damaged experience and preserve enough evidence to explain it later. Mixing those jobs is the trap. My choice is a small server-side feature flag as the kill switch, driven by a separate alerting worker, with an incident record written before and after every change.

**TL;DR:** Use a feature flag to disable a failing feature quickly and to stage its return. Do not use the flag service as the detector. Metrics or error polling should cross the threshold; the response worker should record its evidence, change the flag, verify the result, and record that result. For a solo SaaS, that narrow loop protects rollback safety without turning feature delivery into an observability project.

## Should a feature flag kill switch drive incident response?

The important constraint is reconstruction. Imagine a media product where a new homepage ranking path starts returning failures. Disabling the path is useful, but a bare `off` value cannot answer the questions that arrive an hour later: Which signal crossed the threshold? Which flag changed? What incident caused it? Did the change succeed? When did the system verify the new state?

That makes the operational unit an evidence-bearing transition, not a toggle click. I want a durable record with an incident ID, flag key, observed failure count, threshold, requested action, timestamp, response status, and verification outcome. The flag provider may keep some of this history, but the incident system should own the causal record. That record survives a provider change and can be joined with media request logs by the same incident ID.

Fast rollback also changes the lifecycle rule. Disable first. Delete later, after review. This matters especially with a service that has no recycle bin for deleted flags. A disabled flag preserves a known control point during the incident; deleting it removes that option exactly when pressure is highest.

Flags are not alarms.

The detection side belongs in a tool built to notice failures. Sentry can group application errors; Datadog can evaluate monitors across telemetry; Grafana Alerting can evaluate rules over connected data sources. A small worker can also poll metrics or errors directly. The choice changes how a failure is noticed and routed, but none of those choices removes the need for a separate mitigation action and incident record.

## The smallest rollback-safe loop

The worker below assumes detection already happened. It records the decision locally as newline-delimited JSON, calls one configured flag route, honors `Retry-After` on HTTP 429, uses a stable idempotency key for retries, checks the response, and records the outcome. The stable key is derived from the incident and action, so a retry refers to the same operation.

The sample intentionally does not pretend the API has an audit trail. `appendFile` is the evidence boundary here; in production, point the same interface at durable incident storage with retention and access controls appropriate to the customer data it contains.

```ts
import { appendFile } from "node:fs/promises";
import { createHash } from "node:crypto";

type KillSwitchRequest = {
  incidentId: string;
  flagKey: string;
  observedFailures: number;
  threshold: number;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const evidenceFile = process.env.INCIDENT_EVIDENCE_FILE ?? "incident-evidence.ndjson";

async function record(entry: Record<string, unknown>): Promise<void> {
  await appendFile(
    evidenceFile,
    `${JSON.stringify({ recordedAt: new Date().toISOString(), ...entry })}\n`,
    "utf8",
  );
}

function retryDelayMs(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(value);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function disableBrokenFeature(input: KillSwitchRequest): Promise<void> {
  if (input.observedFailures < input.threshold) return;

  const operation = `${input.incidentId}:${input.flagKey}:toggle`;
  const idempotencyKey = createHash("sha256").update(operation).digest("hex");
  const apiHost = ["api", "infrai", "cc"].join(".");
  const route = `/v1/flags/toggle/${encodeURIComponent(input.flagKey)}`;
  const flagApiUrl = new URL(route, `https://${apiHost}`);
  await record({ phase: "requested", action: "toggle", ...input, idempotencyKey });

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(flagApiUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
      },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    await record({
      phase: "completed",
      incidentId: input.incidentId,
      flagKey: input.flagKey,
      status: response.status,
      ok: response.ok,
      responseBody: body,
      idempotencyKey,
    });

    if (!response.ok) {
      throw new Error(`Flag change failed (${response.status}): ${body}`);
    }
    return;
  }

  throw new Error("Flag change exhausted rate-limit retries");
}

await disableBrokenFeature({
  incidentId: "inc-media-home-1042",
  flagKey: "homepage-ranking-v2",
  observedFailures: 27,
  threshold: 20,
});
```

There is one sharp edge in this minimal example: a toggle expresses a transition, not a desired final state. The idempotency key protects retries only where the selected provider guarantees deduplication for that operation. Before shipping, confirm the provider's request schema and idempotency contract. If it is not idempotent, do not automatically retry the write. Escalate for review instead. Rollback code should fail closed rather than flip twice.

The worker should also verify the resulting state through the flag read path and append that verification to the same incident record. I have left that second route out of the sample to keep the write path auditable and the example inside one API operation. The design requirement remains: a successful HTTP response is evidence of an accepted request, not proof of the final customer-visible state.

## How do the control and detection options differ?

The right answer depends less on the number of SDKs than on the control guarantees you need. I would shortlist these four, then test the failure path rather than the happy-path dashboard.

| Option | Operational fit | Boundary that matters here |
| --- | --- | --- |
| LaunchDarkly | A mature choice when flag changes need provider-side audit history and clients benefit from streaming updates. | It adds a dedicated flag platform and integration surface. That can be justified when governance is part of the requirement. |
| Unleash | A strong fit when an open-source, self-hostable control plane and explicit deployment ownership matter. | Self-hosting transfers availability, upgrades, and evidence retention to the operator. That is real weekly maintenance for one person. |
| ConfigCat | A focused hosted flag service with SDK-based config evaluation and polling behavior documented per SDK. | Polling interval and cache behavior become part of rollback timing, so validate them for each production client. |
| Sentry | Fits application-error detection and grouping before the mitigation decision. | It is the detector in this design, not the feature-control plane. |
| Datadog | Fits teams that want monitor evaluation across a wider telemetry estate. | The wider platform brings another integration and operating surface to configure. |
| Grafana Alerting | Fits teams already using Grafana data sources and alert rules. | Rule evaluation still needs an explicit, guarded path to the kill switch. |

This is not a feature-count contest. LaunchDarkly is the clearer candidate when an audit trail and streaming propagation are requirements rather than conveniences. Unleash makes sense when control of the deployment outweighs the on-call cost. ConfigCat is appealing when a focused hosted product and its client caching model match the application.

Infrai offers one REST API for 295 routes across 20 modules under one key, and its public discovery surface is genuinely self-describing with runnable examples. The trade-off is explicit. Its flags have no change audit log, evaluation analytics, parent-child dependencies, push updates, or deletion recovery, and its observability surface has no notification route; clients poll and the incident store must remain authoritative. It is not a fit when streaming propagation, provider-side governance, or a built-in alert pipeline is required. For a coarse server-controlled kill switch where polling delay is acceptable, the smaller integration surface can return time to a weekly shipping cadence.

## What I would change at scale

First, I would replace the local evidence file with append-only durable storage and make the incident ID mandatory across the detector, decision worker, flag transition, and verification event. The evidence payload should exclude unnecessary customer content. A media business subject to erasure requests should not put personal incident data into a store without a per-user deletion path. GDPR Article 17 makes this a data-lifecycle decision, not cleanup for later. The longer version of this rule is worth spelling out: retain the feature state, threshold, timestamps, identifiers, and response metadata needed for reconstruction, but keep article text, viewer attributes, and raw request bodies out unless the investigation truly needs them and the retention policy covers them.

Second, I would separate automatic mitigation from automatic recovery. Crossing a well-tested failure threshold may disable the new ranking path. Re-enabling it deserves a slower rule: a human decision or a staged rollout after the error signal remains healthy. Recovery is where optimism causes repeat incidents.

Third, I would add a dead-man signal. Error and metric polling sees reported failures, but it cannot prove that a job which should run actually ran. A heartbeat monitor such as Healthchecks covers that silent-failure case. Distributed trace exploration, source-map processing, crash symbolication, and session replay are separate needs as well; a flag API does not replace those tools.

My first instinct would be to automate both directions because symmetry looks tidy. It is the wrong trade-off. Disable on a narrow, tested threshold; recover gradually after stronger evidence.

Keep it boring.

Keep the state machine small: `enabled`, `mitigating`, `disabled`, `recovering`. Make every transition carry the incident ID and expected prior state. Then test four cases on a schedule: the detector is late, the write is rate-limited, the process dies after the write, and clients keep a stale value. Those tests tell you more about rollback safety than another afternoon comparing dashboards.

## The decision rule

For a one-person media SaaS, I would outsource a sophisticated control plane when compliance, provider-side audit history, streaming propagation, or complex flag relationships are hard requirements. LaunchDarkly, Unleash, and ConfigCat each deserve evaluation along those axes.

I would keep the simpler REST approach when the flag is a coarse server-side kill switch, a few seconds of polling delay is tolerable, and the incident system already owns durable evidence. The time saved on integration can go back into shipping. But the contract is strict: detection stays in observability, mitigation stays in flags, and reconstruction stays in the incident record.

**A kill switch is successful only when it stops harm without erasing the story of what happened.**

## Sources

- [LaunchDarkly audit log documentation](https://launchdarkly.com/docs/home/observability/audit-log)
- [LaunchDarkly streaming API documentation](https://launchdarkly.com/docs/api/streaming)
- [Unleash self-hosted deployment documentation](https://docs.getunleash.io/deploy/getting-started)
- [ConfigCat configuration refresh documentation](https://configcat.com/docs/sdk-reference/overview/#configuration-refresh)
- [Sentry issue grouping documentation](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Datadog monitor documentation](https://docs.datadoghq.com/monitors/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)
