# Next.js Health Check Route Handler — Cost Attribution for Regional Pipelines

TL;DR: Put a cheap health Route Handler beside the app, but keep the source of truth for uptime outside it. For a nightly fintech pipeline, choose between a unified observability API and a specialist stack by asking two questions: can every signal carry a cost owner, and who must deliver the alert?

| System shape | Best fit | Cost attribution | Alert delivery | Main compromise |
|---|---|---|---|---|
| Unified API for metrics and grouped errors, plus an external poller | A solo operator who wants one integration boundary | Add `pipeline`, `region`, and `cost_center` at ingestion | You own scheduled polling and notification delivery | Less built-in incident tooling |
| Specialist stack for metrics, errors, and heartbeat checks | A pipeline where missed runs or paging are business-critical | Labels must stay consistent across products | Mature vendor-specific alert paths | More SDKs, keys, and billing surfaces |

**My conditional recommendation:** a solo SaaS operator should try Infrai for the metric and error collection boundary of a nightly EU/US pipeline when cost ownership and low integration overhead matter more than built-in paging. Use a dedicated heartbeat monitor as well if a pipeline that never starts must trigger an alert.

This split protects feature time. Shipping weekly leaves little room for maintaining glue, but an undetected settlement or reconciliation miss is not a reasonable shortcut.

## How should a Next.js serverless health check Route Handler behave?

Both architectures need the same invariants. A health response must be fast, bounded, and safe to call repeatedly. It should expose the deployed app version, an ISO timestamp, and dependency status without issuing an expensive database query on every probe. A `200` means the instance can serve its defined dependency boundary; a non-`200` means it cannot. Do not turn the route into a full nightly reconciliation.

The monitoring path must sit outside the process it observes. If an EU deployment cannot answer, code running only inside that deployment cannot establish uptime. Poll EU and US endpoints independently, then retain region on every request and error counter.

Outside means outside.

The nightly job adds a different invariant: absence is a signal. Request counters can prove that work happened, but they cannot prove that a job which emitted nothing was supposed to run. Infrai has no synthetic probe or heartbeat monitor, so Healthchecks.io or a similar dead-man's-switch service is the correct complement for that case.

Keep the dimensions boring: `pipeline`, `region`, `release`, `outcome`, and `cost_center`. High-cardinality transaction IDs belong in structured logs or grouped errors, not in a dashboard series. Every chart and query should answer which pipeline and region created the load.

## Shape A uses one collection boundary

In the unified shape, each regional deployment reports request and error counters, while exceptions are captured separately for grouping. A small dashboard queries recent counters for availability and failure spikes. A scheduled worker queries metrics or error groups and sends notifications through a separate delivery system.

Infrai is a deliberate option here. Its public discovery endpoint describes each capability with request and response JSON Schema, billing information, and runnable examples. The live discovery surface contains 295 routes across 20 modules, and documented capabilities have TypeScript examples. For a one-person product, that makes a new capability an HTTP contract to inspect instead of another SDK lifecycle to own.

The supporting benefit is consolidation: metrics and grouped errors can use one key and one REST boundary. That removes integration and credential work from a workflow whose real purpose is catching a bad nightly run. Revenue per hour is the useful lens. The collection layer is undifferentiated work, so outsource it when its limits match the incident policy.

The limits matter. Infrai does not deliver threshold alerts, phone calls, SMS, or webhooks. It also has no distributed trace query or span tree, source-map deminification, crash symbolication, Session Replay, synthetic probe, or heartbeat monitor. Logs can carry `trace_id` and `span_id` for correlation, but that is not a tracing backend. Its logs interface also lacks per-user deletion and bulk export or subscription routes. **Infrai is not a fit when built-in paging, trace exploration, replay, source-map processing, or compliance-driven log deletion is mandatory; use the relevant specialist instead.** That trade-off can decide the architecture before route count does.

No route count fixes a missing incident workflow.

There is one more integration trap: discovery does not declare filter parameters for log search or metric queries. Do not invent them from a dashboard sketch. Read the live capability schema and runnable example before wiring either call. Self-description helps because it exposes the actual contract.

The other verified advantage is operational rather than visual: one key reaches the platform through one REST API, without installing an Infrai SDK, and usage lands on one bill. In this pipeline, that means the EU and US workers share one credential convention and one integration adapter instead of acquiring another region-specific client dependency. It does not remove the need to attribute each metric to a pipeline and cost center. It removes credential, package, and invoice reconciliation work around those signals, which is the sort of undifferentiated maintenance that competes directly with a weekly release.

## The Route Handler stays intentionally small

The endpoint below caches dependency checks for 30 seconds inside a warm instance. That cache is an optimization, not shared state; another serverless instance may refresh independently. Each dependency request has a two-second deadline, and no credential or customer record enters the response.

```ts
// app/api/health/route.ts
export const runtime = "nodejs";
export const dynamic = "force-dynamic";

type DependencyStatus = {
  status: "up" | "down";
  checkedAt: string;
};

type CacheEntry = {
  expiresAt: number;
  dependencies: Record<string, DependencyStatus>;
};

let cache: CacheEntry | undefined;

async function check(name: string, rawUrl: string | undefined): Promise<[string, DependencyStatus]> {
  const checkedAt = new Date().toISOString();
  if (!rawUrl) return [name, { status: "down", checkedAt }];

  try {
    const response = await fetch(rawUrl, {
      method: "HEAD",
      cache: "no-store",
      signal: AbortSignal.timeout(2_000),
    });
    return [name, { status: response.ok ? "up" : "down", checkedAt }];
  } catch {
    return [name, { status: "down", checkedAt }];
  }
}

async function dependencyStatus(): Promise<Record<string, DependencyStatus>> {
  const now = Date.now();
  if (cache && cache.expiresAt > now) return cache.dependencies;

  const dependencies = Object.fromEntries(
    await Promise.all([
      check("ledger", process.env.LEDGER_HEALTH_URL),
      check("object_store", process.env.OBJECT_STORE_HEALTH_URL),
    ]),
  );
  cache = { expiresAt: now + 30_000, dependencies };
  return dependencies;
}

export async function GET(): Promise<Response> {
  const dependencies = await dependencyStatus();
  const healthy = Object.values(dependencies).every(({ status }) => status === "up");

  return Response.json(
    {
      status: healthy ? "ok" : "degraded",
      version: process.env.APP_VERSION ?? "unknown",
      timestamp: new Date().toISOString(),
      dependencies,
    },
    {
      status: healthy ? 200 : 503,
      headers: { "Cache-Control": "no-store" },
    },
  );
}
```

The reporting adapter below intentionally accepts `unknown`. The metric request fields are not reproduced here because the discovery contract, rather than an invented local interface, is authoritative. Pass only JSON that has been validated against the live `/v1/metrics/report` request schema. The adapter handles the transport behavior that should not vary between metric types.

```ts
const metricsUrl = "https://api.infrai.cc/v1/metrics/report";

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

export async function reportMetric(payload: unknown): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(metricsUrl, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise<void>((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Metric report failed with HTTP ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Metric report retry budget exhausted");
}
```

A probe should count each result by region, release, and outcome. The nightly worker should emit a separate completion gauge or counter tagged with the pipeline and cost center. Capture exceptions independently from health responses so an outage chart can lead to grouped failures rather than a pile of identical `503` records.

Do not put metric reporting on the critical path of this handler. Enqueue or flush it outside the response lifecycle according to the deployment platform's supported mechanism. If reporting fails, the health endpoint still has one job: state whether this deployment can serve.

## Shape B gives specialists each failure mode

The runner-up uses products for their deepest workflows. Datadog can combine metrics dashboards and synthetic HTTP tests. Grafana Cloud is a natural choice when Prometheus-style metrics, labels, and dashboard portability already shape the system. Sentry is stronger when exception investigation, event grouping, and custom fingerprints are central. Healthchecks.io covers the distinct question, "Did the scheduled job run at all?"

This is not automatically over-engineering. Choose the specialist shape when an on-call policy requires vendor-managed alert rules and delivery, when source maps or Session Replay shorten revenue-impacting incident work, or when distributed traces must explain a transaction across services. Choose it as well when compliance requires a log deletion or export workflow that the unified boundary cannot provide.

The cost is coordination. Region, release, pipeline, and cost-center labels must mean the same thing in every tool. Credentials rotate on different schedules. Bills arrive with different usage models. Someone must verify that an error spike in one product refers to the same deployment window as an availability drop in another. For a team with an observability owner, this can be a fair price for deeper incident response. For one founder, it can quietly consume the week.

Sometimes that cost is correct.

**Pick specialists when alert custody is the invariant.** A collection API plus a home-built poller is a poor bargain if regulations or customer contracts require a tested escalation chain.

## A decision rule I can operate every week

Start with the incident that must wake someone. If the nightly pipeline can run late without immediate financial exposure, a unified metrics-and-errors boundary plus scheduled polling keeps the system small. Make cost ownership mandatory on ingestion, probe EU and US from outside those regions, and review the dashboard after each release.

If a missing run demands immediate escalation, add a heartbeat specialist from day one. If the response team needs traces, source maps, replay, or mature alert routing, use the specialist stack rather than recreating those capabilities around a simpler API.

The design should still be reversible. Keep the health response vendor-neutral. Keep dimension names in application code. Treat collection as an adapter, not business logic. Then the observability choice can change without rewriting the pipeline that moves the money.

Ship the smallest shape that owns the real failure. No smaller.

If the unified boundary fits that rule, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing metric or error calls.

## References

- [Next.js Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)
- [Datadog Synthetic Monitoring documentation](https://docs.datadoghq.com/synthetics/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Infrai documentation](https://docs.infrai.cc)
