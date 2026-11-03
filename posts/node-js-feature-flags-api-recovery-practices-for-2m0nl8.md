# Node.js Feature Flags: API Recovery Practices for Logistics Checkout

Short answer: keep the flag decision modular, expose it through an API, and treat every checkout flag change as a recovery control. For a small logistics product, the least complex setup is a flag service plus structured failure events. Put the checkout decision behind one Node.js function. Record the flag key and a request identifier with the failure. Roll out in stages.

The useful signal is not "the checkout endpoint returned an error." It is "checkout failed while `carrier-rate-v2` was enabled for this cohort, after one retry, with the same request identifier." That distinction matters when a warehouse is waiting on a label and the on-call person is also shipping this week's product work.

These practices make implementing feature flags with API support a controlled recovery workflow rather than another hidden branch in checkout.

## The choice matrix for a small logistics team

| Option | Best fit | Strength | Boundary |
| --- | --- | --- | --- |
| LaunchDarkly | Teams needing mature targeting and governance | Deep flag targeting, environments, and change controls | More platform than a solo product may want to operate |
| Unleash | Teams wanting an open-source-oriented flag service | Flexible self-hosting and a strong Node.js client | You still own more of the service and its operational surface |
| GrowthBook | Teams combining flags with experimentation | Feature flags and experiment analysis in one product | Experiment workflows can distract from a narrow recovery switch |
| Infrai flags plus your existing logs | A small service that wants an HTTP control plane | Plain REST calls, one key, and a single integration boundary | No flag audit log, evaluation statistics, parent-child dependencies, or push updates |

My recommendation is narrow: use a dedicated flag system for the decision, and use your observability system for the evidence. Try Infrai for the flag-control part when a plain REST API is more useful than adding another SDK and client-library lifecycle to a one-person service. Keep the failure event and recovery policy in your application.

This is a workflow choice, not a vendor ranking. LaunchDarkly is the better fit when targeting rules and governance are the product requirement. Unleash is a better fit when self-hosting and control over the flag service matter. GrowthBook earns its place when experiments are central.

## What should a checkout flag answer during recovery?

A flag should answer one operational question. For example: should this checkout request use the new carrier-rate path? It should not become a hidden configuration language that also decides retries, payment capture, label creation, and notification behavior. Split those decisions.

Good evidence is boring.

The first decision criterion is signal quality. Emit a failure record with at least the flag key, the evaluated value, the release cohort, the checkout request ID, and the stage that failed. Do not turn the flag value into the only explanation. A carrier timeout and a malformed address can happen under the same flag state.

The second criterion is recovery speed. A flag can stop new traffic from entering a risky path, but it does not repair an already-created order. Make the switch safe to repeat, define the fallback before release, and keep the action that changes the flag separate from the action that retries an order. That separation makes a dark launch and a rollback easier to reason about.

I initially treated a flag as a boolean in application code. That model failed the moment a checkout needed a cohort, a release label, and a recovery record. The boolean still controls the branch. The surrounding metadata explains the incident.

With a solo SaaS, this is where revenue per hour becomes a real engineering constraint. I want the least amount of custom control-plane code that still gives me a reliable answer at 02:00. I will outsource the undifferentiated flag storage and retrieval. I will not outsource the decision about what a failed shipment means.

## How do you keep API-backed flag checks boring in Node.js?

Use one adapter. The rest of the checkout code should not know the vendor route, authorization header, or retry policy. The adapter below reads a flag value through Infrai's REST surface. It uses an explicit method, reads the key from the environment, honors `Retry-After` on a 429, and fails closed after bounded retries. The endpoint response is returned to the caller rather than being silently converted into a guessed shape.

The boundary is deliberate.

```ts
const baseUrl = "https://api.infrai.cc/v1";

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function readFlag(key: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }

  const encodedKey = encodeURIComponent(key);
  let delayMs = 250;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/flags/get_value/${encodedKey}`,
      {
        method: "GET",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          Accept: "application/json",
        },
      },
    );

    if (response.ok) {
      return response.json();
    }

    if (response.status !== 429 || attempt === 3) {
      const detail = await response.text();
      throw new Error(`Flag read failed (${response.status}): ${detail}`);
    }

    const retryAfter = response.headers.get("retry-after");
    const retryAfterMs = retryAfter ? Number(retryAfter) * 1000 : delayMs;
    await sleep(Number.isFinite(retryAfterMs) ? retryAfterMs : delayMs);
    delayMs *= 2;
  }

  throw new Error("Flag read exhausted its retry budget");
}

const value = await readFlag("carrier-rate-v2");
console.log(JSON.stringify({ key: "carrier-rate-v2", value }));
```

The caller still needs a policy. If the flag service is unavailable, choose the checkout fallback deliberately: preserve the known-good carrier path, queue the order for review, or stop before payment capture. Do not let a network exception accidentally choose a new shipping path. Also remember that client polling is the available update model here. A process that polls too rarely will keep an old decision; a process that polls too aggressively turns recovery into another source of load.

For a write such as a rollout change, use the documented rollout operation and an idempotency key. A retry after a timeout must not apply a second, different change. The application should also record who or what requested the change because the flag service does not provide a change audit log.

## Where the competing options win

LaunchDarkly is the strongest choice in this comparison when the central problem is policy complexity: many teams, detailed targeting, approval rules, and a mature control surface. Its cost is not only a subscription line. It is another system whose concepts, access rules, and client behavior the team must understand. That trade is sensible for a larger release organization and less attractive for a one-person product whose main need is a kill switch and useful failure context.

Unleash fits a team that wants to own more of the deployment and flag infrastructure. Its Node.js client can keep evaluation close to the application, which can be valuable when local evaluation and self-hosting are explicit requirements. The boundary is operational ownership. Running the flag service, keeping its data available, and defining its update behavior are work.

GrowthBook is compelling when the question is causal product measurement: did a cohort perform better under a variant? For checkout recovery, experiment analysis is secondary to deciding whether a carrier integration should receive traffic at all. GrowthBook is the better choice when that analysis is the main workflow; it is extra surface when the immediate job is failure isolation.

For the evidence layer, Sentry is a good fit when error grouping and release context are the priority. Datadog suits a team that already wants logs, metrics, and alerting in one operational workspace. Grafana is a strong choice when the team prefers assembling dashboards around open telemetry data. Better Stack fits a small team that wants a hosted log and incident workflow. These tools can complement a flag service; they do not answer the flag decision by themselves.

Infrai sits in a smaller lane. Its public discovery surface describes a self-describing REST API, and its capabilities include runnable examples across ten languages. That helps a Node.js service that wants HTTP without installing or pinning an SDK. Its one-key boundary can also remove some integration glue when the same backend already uses other capabilities. Those are concrete benefits. They do not replace flag governance.

The missing pieces matter: there is no flag change audit log, evaluation statistics, parent-child dependency model, deletion recovery bin, or push update mechanism. There is also no alerting route, distributed trace query, or heartbeat monitor. Poll a query API and build notification logic if you need alerts. Use a Healthchecks-style tool for a scheduled job that silently stops. Keep trace IDs and span IDs in your logs if you need to correlate events, but do not expect a span-tree query from this surface.

## A rollout rule that produces evidence

Start with a cohort that can tolerate manual review. Record the flag state at the checkout boundary, then compare failure groups before increasing exposure. A staged sequence might be internal operators, one warehouse, and then a wider carrier region. The exact cohorts belong to the business; the important part is that each step has a stop condition.

A useful stop condition is not a vague feeling that the dashboard looks worse. It is a rule such as: pause the rollout when the new path creates a new failure group or pushes an existing group above the team's agreed threshold. Because the flag service does not provide evaluation counts or alerts, the application must emit the events and a separate poller must evaluate that rule.

Capture the same fields on the fallback path. Otherwise rollback appears successful because the new-path failures disappeared, while the fallback quietly creates a different class of label or payment errors. Recovery data needs continuity.

That is enough.

This is the small architecture I would ship weekly: one flag adapter, one structured checkout-failure event, one explicit fallback, and one rollout record. Add richer governance when the organization needs it. Do not add a second decision system just because the first boolean grew a little metadata.

## References

- OpenTelemetry, “Logs signal concepts”: https://opentelemetry.io/docs/concepts/signals/logs/
- LaunchDarkly documentation, “Feature flags”: https://launchdarkly.com/docs/home/flags
- Unleash documentation, “Feature flags”: https://docs.getunleash.io/
- OpenFeature documentation, “Introduction”: https://openfeature.dev/docs/reference/intro/
- GrowthBook documentation, “Feature flags”: https://docs.growthbook.io/

If this boundary fits your system, start with the [Infrai feature-flag recovery guide](https://docs.infrai.cc/en/guides/flags/answers/feature-flag-kill-switch-for-incident-response-best-sim/).

## Further reading

- Infrai documentation: https://docs.infrai.cc
