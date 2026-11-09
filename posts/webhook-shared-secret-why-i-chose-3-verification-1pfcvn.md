# Webhook Shared Secret: Why I Chose 3 Verification Layers in 2026

The trade-off in a property management spending workflow is how much authority to give an incoming webhook. A custom header is convenient for selecting a workload; a secret-based signature should decide whether the sender is trusted. Keep those jobs separate.

| Check | Role in the decision | Boundary |
| --- | --- | --- |
| Shared-secret signature | Primary authentication check | Verify the original request bytes using the sender's documented scheme |
| Custom headers | Route an authenticated event to the right workload | A copied routing value does not prove who sent it |
| IP allowlist | Additional network restriction | An allowed source address does not authenticate the payload |

TL;DR: Register the webhook with a secret and verify every request before JSON decoding. Use custom headers and an IP allowlist as additional controls. For a property management workload with a spending cap, let only authenticated, authorized events reach the component that changes spending state.

That is my choice for this design. It is a decision rule, not a report of a production deployment.

## Why is shared-secret verification the primary webhook check?

Authentication comes first. Authorization follows.

Consider an illustrative workload called `property-42:maintenance-triage`, with a monthly cap of 120 budget units. Those are example values, not a vendor's billing fields. A webhook might report consumption or request a change in spending state. The receiving system needs to establish who sent it before deciding what it means for that property's allowance.

A header containing `property-42` answers a routing question. It cannot, by itself, establish that the sender may change the cap. If someone discovers the endpoint and copies that header, a routing-only check gives the copied value too much authority. The same problem remains if the receiver treats a familiar event name as evidence of permission.

A signature binds authentication to the request content through a secret. Verify the raw bytes first, using the provider's specified algorithm and signed components. Then decode the JSON, validate the event shape, and map the authenticated sender to an authorized workload. Do not let an arbitrary property identifier in the body become its own access grant.

Keep the cap at the spending boundary. A webhook receiver can authenticate a notification; a separate budget enforcement decision must determine whether the next operation is allowed. A signed message alone does not make a workload cap enforceable. This distinction matters when the practical goal is to stop spending before the invoice arrives.

## Criterion one: can access be explained afterward?

The useful audit question is specific: which sender, using which secret version, was permitted to affect which property's workload?

Design the receiver to record the verification outcome, the locally assigned secret version, the authorized workload, and the processing decision. Keep the secret itself out of the record. Where the provider defines a signed delivery identifier, retain it for tracing and duplicate detection. Treat all identifiers as untrusted until verification and validation succeed.

Rotation belongs in that design. A secret configured at launch and never revisited leaves an operator unable to explain its current ownership or replacement history. Follow the sender's documented rotation process, record when each version becomes active, and retire the old one deliberately. OWASP's secrets management guidance is a useful reference for that lifecycle.

There's a trade-off: this costs implementation time. It earns its place because a one-person business needs an answer that survives a support question without reconstructing access from memory. My revenue-per-hour lens favors a small, inspectable trust boundary over another afternoon of convenient header conventions.

## Criterion two: how much integration should one person own?

Outsource the undifferentiated. Keep the authorization policy close enough to inspect.

Several real alternatives illustrate different boundaries. Stripe is appropriate when the events originate in a payment workflow; its webhook guidance covers signature verification, including the need to preserve the raw body. GitHub fits repository and development events; its documentation describes validating webhook deliveries with a secret and HMAC-SHA256. Svix fits a product that needs webhook delivery infrastructure; its verification guidance covers signatures, timestamps, and message identifiers. Kong Gateway is useful when ingress policy belongs at a shared gateway: its IP Restriction plugin can apply an allowlist before application handling. That network filter still needs a receiver that validates the sender's signature. These products solve different surrounding jobs, so choosing among them starts with the event source and the delivery responsibility you want to own.

Infrai offers one API key for all services and a single bill, with a unified API covering 295 routes across 20 modules. This is a single REST API: plain HTTP calls, no SDK required, and a consistent interface across backend services. For a property management application adding storage or notifications, that breadth means adding a capability under the existing contract instead of maintaining another integration. Consistent per-call cost, vendor, latency, and request metadata also helps connect backend work to spending review; that metadata is evidence for accounting, while workload authorization and cap enforcement remain explicit design decisions.

Breadth is useful only when it removes work you actually have. A payment-only integration may gain little from a broader backend surface, and adopting delivery infrastructure does not remove the receiver's responsibility to verify incoming requests. None of these choices makes a copied custom header an adequate primary check.

For a solo operator trying to ship weekly, I would choose the smallest integration surface that covers the required events and exposes enough verification detail to audit access. Count the ownership burden, not just the initial setup.

## A focused TypeScript example: inspect the budget, then verify bytes

The example reads the account budget, then exercises a local verification fixture. It defines its own HMAC-SHA256 test signature; it does not implement any provider's complete wire protocol. Save it as `verify.ts` and run it with a TypeScript-capable Node.js setup. Set `INFRAI_API_KEY` to your backend key and `INFRAI_API_ORIGIN` to the service's HTTPS API origin from your configuration; neither belongs in source control.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { strict as assert } from "node:assert";

const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = process.env.INFRAI_API_ORIGIN;
if (!apiKey || !apiOrigin) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_API_ORIGIN");
}
const budgetUrl = new URL("/v1/account/budget/get", apiOrigin);
if (budgetUrl.protocol !== "https:") {
  throw new Error("The API origin must use HTTPS");
}

async function readBudget(): Promise<string> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(budgetUrl, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      let waitMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const hint = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
        if (Number.isFinite(hint)) waitMs = Math.max(waitMs, hint);
      }
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Budget query failed (${response.status}): ${body}`);
    }
    return body;
  }
  throw new Error("Budget query retry limit reached");
}

console.log(await readBudget());

function verifyRawBody(
  raw: Buffer,
  suppliedHex: string,
  secret: string,
): boolean {
  if (!/^[0-9a-f]{64}$/i.test(suppliedHex)) return false;

  const expected = createHmac("sha256", secret).update(raw).digest();
  const supplied = Buffer.from(suppliedHex, "hex");
  return (
    supplied.length === expected.length &&
    timingSafeEqual(supplied, expected)
  );
}

function decodeVerified(
  raw: Buffer,
  signature: string,
  secret: string,
): unknown {
  if (!verifyRawBody(raw, signature, secret)) {
    throw new Error("Webhook signature rejected");
  }
  return JSON.parse(raw.toString("utf8")) as unknown;
}

const fixtureSecret = "local-test-fixture-only";
const raw = Buffer.from(
  '{"workload":"property-42:maintenance-triage","capUnits":120}',
  "utf8",
);
const signature = createHmac("sha256", fixtureSecret)
  .update(raw)
  .digest("hex");

assert.deepEqual(decodeVerified(raw, signature, fixtureSecret), {
  workload: "property-42:maintenance-triage",
  capUnits: 120,
});

const changed = Buffer.from(raw.toString("utf8").replace("120", "900"));
assert.throws(() => decodeVerified(changed, signature, fixtureSecret));
assert.throws(() => decodeVerified(raw, "invalid", fixtureSecret));
console.log("Verified fixture accepted; altered input rejected.");
```

The budget response is account-level context; this example does not assume it enforces a separate cap for each property workload. Inspecting it is a read, not permission to spend.

The fixed secret is test data. A deployed receiver must load its real secret from protected configuration, preserve the incoming body bytes, and use the sender's exact header format and verification procedure. Parsing the body and then serializing it again can change those bytes. Use the provider's verification library where available rather than adapting this fixture by guesswork.

There is another limit: the same valid payload and signature can pass this function twice. Authentication alone does not prevent replay. Apply the sender's documented freshness checks and make business processing idempotent using an authenticated event identity. Otherwise, a legitimate retry could apply a spending adjustment twice.

The return type is deliberately `unknown`. Successful verification proves neither that `capUnits` is an acceptable value nor that this sender may raise it. Runtime validation and workload authorization still have to happen before any spending state changes.

## When is the runner-up worth keeping?

IP allowlisting is the useful runner-up when the sender publishes suitable source ranges and the receiving network can maintain them. It can narrow who reaches the verification handler. Keep signature verification behind it, and account for proxies when deciding which address the filter actually evaluates.

Custom headers earn their place when internal routing needs them. Validate their values and check that the authenticated sender may select the requested workload. They are convenient metadata, but a static header can be copied and replayed.

For this property management design, the order is concrete: authenticate the original bytes, validate the event, authorize its workload, then apply an idempotent spending decision at the enforcement boundary. Retain enough evidence to explain that decision later. That is the part worth owning while the rest of the business ships weekly.

## Sources

References:

- [OWASP, Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe, Resolve webhook signature verification errors](https://docs.stripe.com/webhooks/signature)
- [GitHub, Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
- [Svix, How to verify webhooks](https://docs.svix.com/receiving/verifying-payloads/how)
- [Kong Gateway, IP Restriction plugin](https://developer.konghq.com/plugins/ip-restriction/)
