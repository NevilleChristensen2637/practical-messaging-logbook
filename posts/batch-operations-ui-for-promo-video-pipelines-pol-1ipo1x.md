# Batch Operations UI for Promo Video Pipelines — Polling, Cancellation, and Terminal States

A batch console for game promo videos is a bandwidth problem before it is an API problem. If the UI guesses what is happening, one stuck tab can burn an afternoon of support time. I would drive the screen from the batch status, allow cancellation only while the operation can still change, and stop polling as soon as the status is terminal.

Short answer: persist the batch and asset IDs, validate each stage before starting the next one, make retries idempotent in your application, and treat `succeeded`, `failed`, and `cancelled` as terminal states.

That rule is deliberately boring. Boring is good when a solo founder has to ship a weekly release and still answer players who cannot find a trailer.

## Why the console needs a state machine

The tempting implementation is a button that starts work and a spinner that calls an endpoint every few seconds. It looks fine in a demo. It falls apart when a browser is closed between two transformations, when a user clicks Cancel twice, or when a derivative is created but the UI never records its relationship to the source.

I model the workflow as explicit stages: `submitted`, `running`, `cancelling`, then one of the terminal outcomes. Each row stores the batch identifier and the identifiers for the source and derivative assets. A refresh reconstructs the same view from those records; it does not depend on a JavaScript timer surviving a tab.

The stage boundary matters. Before handing a generated frame to a crop or conversion step, the worker checks that the previous result exists and is in the expected state. If that check fails, the next transformation is not attempted. The console can show a useful reason instead of presenting a green thumbnail that has no durable source.

Lineage is part of the product, too. Keep `source_asset_id`, `derivative_asset_id`, the batch id, and the transformation name together. That gives support a path to inspect, gives cleanup a safe set of candidates, and makes an audit of a campaign possible weeks later.

One short sentence can save a lot of polling.

Ship weekly.

Keep it observable.

## How should polling, cancellation, and terminal-state handling work?

Polling is a lease on attention, not the source of truth. The worker and the API own the state; the browser only observes it. Start with a modest interval, add a little jitter so a campaign of tabs does not wake at once, and stop scheduling the next request when a terminal state arrives.

Cancellation follows the same rule. Show the action for states where cancellation is meaningful, such as `submitted` or `running`. Once the operation is `succeeded`, `failed`, or `cancelled`, hide or disable the action and render the recorded outcome. A double click is still possible, so the server call needs an application-level idempotency key derived from the batch id and the requested action.

Here is the small client I would put behind a console row. It uses the documented batch status and cancellation paths, retries a rate-limit response with the server's `Retry-After` hint when available, and surfaces non-success responses instead of turning them into an endless spinner. The base URL is injected at deploy time, which keeps a local test, a staging account, and production from sharing a baked-in host.

```ts
const TERMINAL = new Set(["succeeded", "failed", "cancelled"]);
const CANCELLABLE = new Set(["submitted", "running"]);

type BatchStatus = {
  status: string;
  [key: string]: unknown;
};

function wait(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function request(
  url: string,
  method: "GET" | "POST",
  idempotencyKey?: string,
): Promise<BatchStatus> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const headers: Record<string, string> = {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    };
    if (idempotencyKey) headers["Idempotency-Key"] = idempotencyKey;

    const response = await fetch(url, {
      method,
      headers,
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await wait(delay);
      continue;
    }

    const body = (await response.json()) as BatchStatus;
    if (!response.ok) {
      throw new Error(`Batch request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Rate limit did not clear after five attempts");
}

export async function watchBatch(batchId: string): Promise<BatchStatus> {
  const apiBase = process.env.INFRAI_API_BASE;
  if (!apiBase) throw new Error("INFRAI_API_BASE is required");
  let intervalMs = 1500;
  for (;;) {
    const result = await request(
      `${apiBase}/v1/image/batch/status/${batchId}`,
      "GET",
    );
    if (TERMINAL.has(result.status)) return result;
    await wait(intervalMs + Math.floor(Math.random() * 400));
    intervalMs = Math.min(intervalMs * 2, 10000);
  }
}

export async function cancelBatch(batchId: string, currentStatus: string) {
  if (!CANCELLABLE.has(currentStatus)) {
    return { status: currentStatus, skipped: true };
  }
  const apiBase = process.env.INFRAI_API_BASE;
  if (!apiBase) throw new Error("INFRAI_API_BASE is required");
  return request(
    `${apiBase}/v1/image/batch/cancel/${batchId}`,
    "POST",
    `batch:${batchId}:cancel`,
  );
}
```

The sample intentionally keeps the UI policy visible. If the page reloads, call `watchBatch` with the persisted id. If cancellation returns a state that is already terminal, render that state and stop; the client does not invent a second outcome. For a real queue, the consumer should also deduplicate by the same application key because standard queues are at-least-once systems.

## Choosing the execution boundary

The API is only one part of the decision. A one-person SaaS usually has four plausible shapes for this workflow:

| Option | Where it fits | Trade-off for a promo-video console |
| --- | --- | --- |
| Self-hosted workers | You need custom codecs, private GPUs, or full control of the queue | Maximum control, but you own capacity, patching, and every retry path |
| AWS Elemental MediaConvert | Your pipeline already lives in AWS and needs broadcast-grade media jobs | Deep media controls, with more account, IAM, and service plumbing |
| Cloudinary transformations | You want hosted asset transformations and delivery around a media library | Fast to integrate, but the workflow model is shaped by its asset platform |
| Mux Video | Video ingest, playback, and observability are the main product surface | Strong video workflow, less suited to a broad set of non-video derivatives |
| imgix | Image delivery and URL-based transformations are already your center of gravity | Excellent delivery primitives, but a multi-stage batch state machine remains your job |
| Cloudflare Images | You want managed image storage and delivery close to a Cloudflare edge | Convenient image operations, with less room for a bespoke video-generation queue |
| Infrai | You want a plain HTTP boundary for several backend capabilities | One REST API and one key reduce SDK and credential coordination; you still need your own state table and product-specific policy |

The last row is not a universal recommendation. Its useful advantage here is the plain REST interface: a TypeScript worker, a small Go service, or a test script can make the same HTTP call without installing a client library. That can be a real revenue-per-hour win when the console also touches storage or another backend capability, because the integration boundary stays consistent.

The catch is scope. Pick a focused media service when you need its specialized encoding controls, playback analytics, or an existing team of operators. Choose self-hosted workers when data residency or custom GPU scheduling outweighs the maintenance cost. Your mileage may vary; I am not sure a single boundary is worth changing for a mature pipeline that already has reliable queue semantics.

## What I would change at scale

At small volume, the persisted row plus a status poll is enough. At campaign volume, I would move polling off the browser: a worker records status transitions, the UI reads a local projection, and a push notification wakes the page only when there is a change. The terminal-state rule stays exactly the same.

I would also add an outbox for lineage events. A successful derivative write emits one event containing the source and derivative identifiers; cleanup and support tools consume that event instead of scraping logs. Retries then become a property of each stage, not a guess made by whoever happens to have the tab open.

Do not optimize this by adding more buttons. Optimize for a clear record of what happened.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://docs.aws.amazon.com/mediaconvert/latest/ug/what-is.html
- https://cloudinary.com/documentation/image_transformations
- https://www.mux.com/docs/guides/video
