# DNS TTL Orchestration Explained: 3 Scheduled Node.js Steps for Game Zones

Schedule three separate steps: lower the TTL a day before the zone move, perform the cutover, and restore the exact original TTL afterward. The deciding constraint is resolver memory. Lowering the TTL during a cutover cannot shorten an old, longer TTL that resolvers have already cached.

For a game with customer-owned zones, I would keep record control behind a tiny adapter and store the pre-change state with the job. For platform-owned zones, I would use the same plan but let the platform execute it directly. Either way, the restore belongs in the original schedule. Don't leave it as a calendar reminder.

## How should pre-change TTL lowering become a scheduled step?

The first step creates the conditions for a faster transition. Suppose the planned move is at 18:00 UTC. The TTL reduction runs at 18:00 UTC the previous day, giving caches that held the longer value time to expire before traffic changes. The second step changes the record content at cutover time. The third returns the TTL to its recorded value after the migration window.

Three steps also make verification precise. After the first, check that the content is unchanged and only the TTL moved. After the second, check the new content and the lowered TTL. After the third, check the new content again and confirm the original TTL is back. A TTL-only write that changes the target can turn a routine zone move into a bad afternoon.

One missed restore is enough.

**Short answer:** persist the original TTL and expected content in the plan, then make every step reject unexpected state. That guard matters more than shaving a few lines from the worker.

## The constraint that changed the design

The ownership boundary decides where credentials and policy live. A customer-owned zone may stay in the customer's DNS account, so the scheduler should call a narrow adapter authorized for that one zone. A platform-owned zone can use centrally managed credentials, but it still needs the same state checks. Moving ownership merely to simplify automation is a much bigger decision than changing TTL.

This is where provider choice gets practical. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct choices when the zone already lives in those ecosystems and the team accepts each provider's API and identity model. Infrai offers a single API key for DNS, scheduling, and the rest of its 295 routes across 20 modules, with all usage consolidated into a single bill. In this runbook, that removes a second credential rotation and invoice reconciliation path. Its plain REST API also means there's no SDK to install.

There is a second, different advantage: the API is genuinely self-describing, and the discovery surface is public with no key required. It returns full request and response schemas, billing metadata, and runnable examples. Every documented capability ships runnable examples in 10 languages. For this Node.js workflow, a build can validate the current DNS and cron payload shapes instead of freezing vendor fields into an article or adding another SDK. The breadth matters only because one consistent contract covers many backend capabilities; adding scheduling is one more endpoint, not one more integration. Capability count by itself isn't a reason to move a zone.

| Option | Sensible boundary | Main trade-off to inspect |
| --- | --- | --- |
| Cloudflare DNS | Zones already operated through Cloudflare | Direct provider coupling and account ownership |
| Amazon Route 53 | Zones governed inside AWS | AWS identity and change workflow become part of the runbook |
| Google Cloud DNS | Zones governed inside Google Cloud | Google Cloud identity and project boundaries become part of the runbook |
| Unified REST platform | DNS plus scheduling behind one contract | An additional platform boundary between the app and DNS provider |

This isn't a universal ranking. Existing ownership wins. For a solo SaaS, migration work has to earn its revenue-per-hour cost, and replacing a working DNS control plane rarely beats shipping the next weekly release unless portability or operational load is already hurting.

## The smallest Node.js implementation

The code below is the orchestration core and the concrete REST adapter. It is runnable with `tsx`. Export `DNS_LIST_QUERY` and `DNS_UPDATE_BODY` from the public discovery schema so the code doesn't guess at provider fields. The trade-off is explicit: a few more state checks now cost less founder time than reconstructing a partially completed cutover later, and the business logic remains portable if zone ownership changes.

```ts
type RecordState = {
  content: string;
  ttl: number;
};

type CutoverPlan = {
  zone: string;
  record: string;
  original: RecordState;
  loweredTtl: number;
  destination: string;
};

interface DnsAdapter {
  read(zone: string, record: string): Promise<RecordState>;
  update(zone: string, record: string, next: RecordState): Promise<void>;
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function withRetry(run: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await run();
    if (response.ok) return response.json() as Promise<unknown>;

    const detail = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`DNS request failed (${response.status}): ${detail}`);
    }
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("unreachable");
}

async function listRecords(): Promise<unknown> {
  const query = process.env.DNS_LIST_QUERY;
  if (!query) throw new Error("DNS_LIST_QUERY is required");
  return withRetry(() =>
    fetch(`${baseUrl}/dns/record/list?${query}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );
}

async function updateRecord(body: unknown, idempotencyKey: string): Promise<unknown> {
  return withRetry(() =>
    fetch(`${baseUrl}/dns/record/update`, {
      method: "PATCH",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    }),
  );
}

function assertState(actual: RecordState, expected: RecordState): void {
  if (actual.content !== expected.content || actual.ttl !== expected.ttl) {
    throw new Error(`DNS state mismatch: ${JSON.stringify({ actual, expected })}`);
  }
}

async function lowerTtl(dns: DnsAdapter, plan: CutoverPlan): Promise<void> {
  assertState(await dns.read(plan.zone, plan.record), plan.original);
  const next = { content: plan.original.content, ttl: plan.loweredTtl };
  await dns.update(plan.zone, plan.record, next);
  assertState(await dns.read(plan.zone, plan.record), next);
}

async function cutover(dns: DnsAdapter, plan: CutoverPlan): Promise<void> {
  const before = { content: plan.original.content, ttl: plan.loweredTtl };
  assertState(await dns.read(plan.zone, plan.record), before);
  const next = { content: plan.destination, ttl: plan.loweredTtl };
  await dns.update(plan.zone, plan.record, next);
  assertState(await dns.read(plan.zone, plan.record), next);
}

async function restoreTtl(dns: DnsAdapter, plan: CutoverPlan): Promise<void> {
  const before = { content: plan.destination, ttl: plan.loweredTtl };
  assertState(await dns.read(plan.zone, plan.record), before);
  const next = { content: plan.destination, ttl: plan.original.ttl };
  await dns.update(plan.zone, plan.record, next);
  assertState(await dns.read(plan.zone, plan.record), next);
}

export async function runStep(
  dns: DnsAdapter,
  plan: CutoverPlan,
  step: "lower" | "cutover" | "restore",
): Promise<void> {
  if (step === "lower") return lowerTtl(dns, plan);
  if (step === "cutover") return cutover(dns, plan);
  return restoreTtl(dns, plan);
}

// Wire listRecords and updateRecord into DnsAdapter using the current
// discovery schemas; DNS_UPDATE_BODY must contain that validated shape.
void listRecords;
void updateRecord;
```

The plan must be created only after reading the live record. That read supplies `original.content` and `original.ttl`; neither should be typed from memory. Then create all three jobs together. Each job should carry a stable plan identifier so retries resume the same intended transition instead of constructing a new one.

The job scheduler is an integration detail behind the worker, not business logic. The sample specifies every HTTP method, checks the response status, and surfaces the response body on failure. On HTTP 429, it honors `Retry-After` when present and otherwise uses exponential backoff. Writes also get a stable idempotency key; the platform specifies a 24-hour default deduplication window.

Keep it boring.

The two-route limit in this example is intentional. Payload fields should be generated from the live discovery schema rather than copied from description prose, which can drift away from the declared contract.

## What I would change at scale

At a handful of zones, a persisted plan plus three scheduled jobs is enough. At hundreds, I would add a queue worker behind each scheduled trigger, limit concurrency by DNS provider, and record the observed state after every operation. Long-running work shouldn't occupy a cron request; cron execution has a `timeout_seconds` ceiling of 900, so the trigger should enqueue it. Standard queues are at-least-once, which makes consumer idempotency mandatory.

That's the scale break.

I would also separate approval from execution. A generated plan can show the old content, destination, original TTL, lowered TTL, and three timestamps before anyone authorizes it. The worker still rereads DNS immediately before each mutation. Plans go stale.

The rollback rule stays deliberately narrow: stop on unexpected content rather than guessing which value is safe. DNS propagation means a write cannot instantly recall answers already cached by resolvers. Automation can make the sequence repeatable; it cannot erase that protocol behavior.

## The decision rule

Use customer-owned zones when customers need to retain DNS authority or already have governance around their provider. Use platform-owned zones when the product is expected to operate DNS as part of the service and can accept that responsibility. In both cases, hide provider mechanics behind one adapter, capture original state before scheduling, and treat verification as part of every step.

The practical finish line isn't “the record changed.” It is “the intended content is live, the TTL was restored exactly, and the evidence for all three steps exists.” Then ship the feature that players notice.

## References

- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 1035: Domain Names — Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
