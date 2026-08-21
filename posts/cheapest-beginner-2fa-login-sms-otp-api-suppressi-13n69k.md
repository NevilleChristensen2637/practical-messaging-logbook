# Cheapest Beginner 2FA Login: SMS OTP API Suppression and Status Polling

Short answer: for a beginner US/EU SaaS, put a managed SMS OTP API behind a small server-side interface, keep suppression and login state in your own database, and run status polling in one background worker. Choose direct messaging only when custom routing or message control earns enough revenue per engineering hour to justify owning the verification machinery.

| Option | What the application owns | Best fit | Main limitation |
|---|---|---|---|
| Managed verification | Login policy, suppression, audit history, recovery | A small team shipping weekly | Less control over code generation and message flow |
| Direct SMS transport | Codes, expiry, attempts, abuse controls, delivery mapping | An established authentication system with unusual rules | A larger security and operations surface |
| Non-SMS authentication | Enrollment, recovery, channel policy | Users who can reliably use another factor | Different support and onboarding work |

**Default to the first row, but preserve the third as an explicit recovery path.** This isn't a universal product recommendation. It is an ownership decision: outsource undifferentiated delivery work while keeping the policy that shapes account access inside the application.

## What should a beginner US/EU SaaS require from a 2FA SMS OTP API?

The first criterion is a clean ownership boundary. The browser should ask the application server to begin a login challenge. The server checks eligibility and suppression, creates an internal attempt, and calls a narrow transport interface. A privileged transport credential never belongs in browser code. Product state should not depend on a provider's response object surviving unchanged forever.

Treat an OTP login as a state machine, not a send button. A useful internal model separates `created`, `suppressed`, `submitted`, `delivered`, `verified`, `expired`, and `failed`. Submission only means that the transport request reached an accepted application state; it does not prove that a person received a code. Verification is the transition that can authorize a session. Delivery information can guide support and retries, but it cannot stand in for verification.

That distinction closes several ordinary failure paths. Consider a test with two browser tabs and one login intent. Tab A begins attempt `login-42`; before its response arrives, Tab B asks for a resend using the same intent. Both requests pass a read-only eligibility check, so a later database reservation must decide the winner. Tab A reserves the intent and submits the message. Tab B finds the reservation and returns the same safe product state without submitting another message. A delayed worker then observes an older transport event, but its conditional update cannot change a newer or completed attempt. Finally, the test verifies the code from Tab A and asserts that only verification, not submission or delivery, authorizes the session. This is a small fixture — no invented production drama required — yet it exercises concurrency, idempotency, stale work, and the exact security boundary in one readable sequence. None of those rules should be hidden in a messaging callback.

The second criterion is operational legibility. Preserve an internal attempt ID, the opaque transport reference, timestamps, the suppression reason, and every accepted state transition. Do not log an OTP or a full phone number. A support view needs to answer a smaller set of questions: Was the request suppressed? Was it submitted? Did polling reach a terminal result before the product deadline? Was another factor offered? This is enough to investigate the flow without turning authentication logs into a second credentials store.

US/EU scope also deserves an explicit policy table in application code. "EU" isn't a routing mode, and a launch region isn't proof that every destination has equal operational support. Start with the countries the business can actually support, normalize destinations before policy checks, and keep display formatting separate. I'm not sure the same fallback policy fits every customer base; enrollment completion and support evidence should settle that choice.

Keep it boring.

## Suppression and polling determine the operating burden

Suppression is a decision, not one boolean. The application may decline a send because an equivalent challenge is still active, an account or destination crossed a product threshold, a recovery flow is required, or the requested destination is outside the supported policy. Store a stable internal reason code. Return a deliberately bland response to the browser so an attacker cannot use detailed denial text to probe accounts or policy thresholds.

The important ordering is check, reserve, then send. The server evaluates suppression and atomically reserves the login intent before calling the transport. If two requests race, only one reservation wins. A successful submission attaches the transport reference to that attempt. A declined request does not enter the polling queue. This ordering avoids paying for duplicate work and, more importantly for a one-person SaaS, keeps duplicate state from consuming a Friday that should have gone to a customer-visible release.

Status polling belongs in a worker, never in every open browser tab. The worker owns a bounded schedule, maps transport-specific values into the application's small state set, and stops at a terminal observation or the application's deadline. The pending page reads application state. It does not poll the transport directly. That arrangement puts retry volume, credentials, mapping rules, and logs in one place.

Use unequal intervals and add jitter if many attempts could become eligible together. The exact timing is a product and transport choice, so I would not copy arbitrary numbers from an example article. The invariant matters more: one durable job owns each attempt, each observation is safe to repeat, and an observation can advance only the attempt it names. If the worker receives an unknown upstream value, it records the sanitized value for investigation and leaves product code inside the known union.

HTTP handling needs one easily missed check. MDN documents that `fetch()` resolves after the server responds with headers, including when the HTTP status is an error; a rejected promise is therefore not the only failure signal. Inspect `response.ok` before parsing a success payload. In a test fixture, make a `429` response resolve normally and assert that the adapter returns a typed rejection instead of manufacturing a provider reference. That single case catches the common assumption that `catch` handles every unsuccessful request.

Cost comparisons should include messages, abuse, support, and maintenance time. "Cheapest" cannot be reduced to a public per-message figure that varies by destination and changes independently of the application. I would review attempted sends by country, suppression reason, time to terminal observation, verification completion, and fallback use. Those measurements show where engineering work can change an outcome. A price table alone does not.

Revenue per hour wins.

## How does a narrow TypeScript boundary keep login code portable?

The application-facing interface should express application concepts. It should not reproduce every transport field. The example below omits queue and database implementations on purpose; those are durable infrastructure around the function, not details to fake in a snippet.

```ts
type DeliveryState = "submitted" | "delivered" | "failed" | "unknown";

type SubmitResult = {
  transportRef: string;
  state: "submitted";
};

interface OtpTransport {
  submit(input: {
    destination: string;
    attemptId: string;
  }): Promise<SubmitResult>;

  readStatus(transportRef: string): Promise<DeliveryState>;
}

type PollableAttempt = {
  id: string;
  transportRef: string;
  deadlineMs: number;
};

type Observation = {
  state: DeliveryState;
  shouldPollAgain: boolean;
};

async function observeAttempt(
  transport: OtpTransport,
  attempt: PollableAttempt,
  nowMs: number,
): Promise<Observation> {
  if (nowMs >= attempt.deadlineMs) {
    return { state: "unknown", shouldPollAgain: false };
  }

  const state = await transport.readStatus(attempt.transportRef);
  const terminal = state === "delivered" || state === "failed";

  return {
    state,
    shouldPollAgain: !terminal,
  };
}
```

The database update after `observeAttempt` should be conditional on the attempt ID and its current state. That is where stale work loses harmlessly. The scheduler can enqueue another observation only when `shouldPollAgain` is true and the attempt is still current. Login code sees this normalized record; it never imports a transport SDK type.

Test the boundaries that create support tickets: two simultaneous begins, repeated submit after a local timeout, polling the same reference twice, an out-of-order observation, an unfamiliar status, expiry before the worker runs, and successful verification while a delivery observation is still pending. Also test log output. A passing behavior test that leaks a code in an error object is not a passing authentication test.

I would keep one fake transport for deterministic state-machine tests and a separate adapter contract suite for the real HTTP implementation. The fake proves product rules. The contract suite proves request serialization, `response.ok` handling, payload validation, and status mapping. This split is quick enough for every pull request and specific enough to say which boundary broke.

Ship weekly. The narrow interface makes that cadence realistic because replacing transport details does not require rewriting login policy.

## When should the runner-up or a non-SMS factor lead?

Direct SMS transport is the runner-up when the business already owns a mature verification service or the message lifecycle itself is differentiated. Country-specific routing, custom risk decisions, or a carefully controlled code lifecycle can justify the extra work. The catch is clear: the application then owns generation, storage, expiry, attempt limits, replay resistance, abuse handling, and more delivery operations. It is not suitable merely because one transport quote looks lower.

A non-SMS factor should lead when the audience can enroll and recover it reliably and the account risk model supports that choice. It should also remain available as an explicit fallback when SMS policy suppresses a request or a destination is outside the supported footprint. Don't switch channels silently. Ask the user to choose an allowed recovery path, then record the transition in the audit history.

Email login needs a product-owned success event. Apple's Mail Privacy Protection can prevent senders from seeing whether a recipient opened a message and can hide the recipient's IP address. An open signal is therefore unsuitable as proof that a login link was received or used. Successful redemption of a signed, expiring link at the application server is the authentication event; mail telemetry is operational context.

There is no permanent winner. Start with the category that minimizes security-sensitive ownership, keep policy and audit state portable, and promote the runner-up only when observed requirements justify its larger surface. For a solo operator, the best stack is the one that protects login integrity while leaving enough uninterrupted time to ship the work customers buy.

## References

- Apple, "Use Mail Privacy Protection on iPhone": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- MDN Web Docs, "Fetch API": https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
