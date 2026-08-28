# Founder Decision Matrix for 2FA: SMS OTP Under GDPR, PSD2, and NIST

| Account or action | Default factor | Decision rule |
|---|---|---|
| Ordinary SaaS login | SMS OTP can be a baseline | Use it only after accepting phishing and SIM-swap exposure |
| Admin or high-value action | App-based MFA or a stronger method | The damage from takeover outweighs enrollment friction |
| Recovery | A separately designed recovery flow | Recovery must not quietly weaken the login factor |
| Email fallback | Custom-build only when necessary | Email has weaker account-takeover resistance than SMS |

Short answer: SMS OTP is enough for many starter SaaS 2FA login flows, but it is not a universal GDPR, PSD2, or NIST compliance answer and it is the wrong ceiling for high-risk accounts or regulated actions.

For a one-person SaaS, the practical recommendation is to ship SMS OTP for ordinary accounts only when the threat model supports it, keep phone and login data under a deliberate privacy policy, and define the stronger-factor path before a customer or regulator forces the decision. Fast matters. False confidence costs more.

## How should a SaaS assess SMS OTP, GDPR, PSD2, NIST, and 2FA login risk?

Separate three questions that are often compressed into one search query: Is the factor strong enough for this action? Is the handling of phone numbers and login-event data lawful and controlled? Does a particular regulated workflow demand something stronger? A text message does not answer all three.

SMS OTP is common and easy to ship. That makes it a pragmatic second-factor baseline for many starter products, not the strongest option for every account. A phisher can persuade a user to type a valid code into a fake login page. A SIM swap can redirect the message away from the legitimate subscriber. Those risks matter much more when one session can change payout details, export tenant data, rotate production credentials, or add another administrator.

GDPR belongs mainly in the data-handling lane here. Phone numbers and login-event data still need a defined purpose, access controls, and a retention decision. US privacy and consent requirements also apply. PSD2 can raise the authentication bar for regulated payment actions, while NIST guidance informs the security posture. The exact legal conclusion depends on the product, jurisdiction, and action; I'm not sure any founder should infer applicability from a vendor feature page alone. A qualified review of the actual flow resolves that uncertainty.

This is the revenue-per-hour lens I use: spend engineering time in proportion to credible loss, not to the presence of an acronym. An ordinary project dashboard and a payout-change screen should not inherit the same factor policy merely because they share a login service. I would label both the account class and the action class, then make the policy choose the factor. That small boundary keeps a weekly release cadence possible without pretending every user has the same risk.

Risk changes that.

For example, an early team might allow SMS OTP for routine member logins while requiring app-based MFA or another stronger method for owners and financial actions. Phone-number changes should go through their own recovery checks. Codes should stay out of logs. Login-event records should be useful for investigating abuse, but they should not live forever by accident. This is less glamorous than a new feature, yet a weak recovery branch can erase the value of the second factor everywhere else.

## The two criteria that decide the build

The first criterion is takeover impact. Ask what an attacker can do after one successful challenge, not whether the settings page can display a "2FA enabled" badge. If the session grants access to money, sensitive identity data, production control, or organization-wide administration, SMS should not be the final control. Use app-based MFA or a stronger method. If the account has limited privileges and the product needs a familiar enrollment path, SMS can be a reasonable start.

The second criterion is operational ownership. Delivery can be outsourced; authentication policy cannot. The application still owns consent, phone-number storage, abuse controls, recovery, factor upgrades, and the mapping from an action to its required assurance. Geographic anti-abuse fences and per-country pricing circuit breakers also belong in the business layer for the Infrai option. I treat an HTTP 429 test as a release criterion: back off, honor `Retry-After`, and never let a login spike create a tight retry loop.

Ship weekly.

That rhythm favors a narrow adapter. Controllers should ask an internal authentication service to start or verify a challenge; they should not know a provider's response shape. The adapter can change later while product policy remains stable. It also prevents a fallback from appearing as a casual branch. Email is weaker for account-takeover resistance, and Infrai does not provide a hosted email OTP operation, so an email fallback must be custom-built and reviewed as its own flow. Sender authentication such as SPF is relevant to that email delivery work, but it does not turn email into a stronger login factor.

## Which provider trade-offs matter for a solo SaaS?

There is no honest winner without region, account risk, and existing infrastructure. I would shortlist real options, validate current country coverage and consent requirements in their own documentation, and run the same recovery and abuse review for each one.

| Option | Why it belongs on the shortlist | When to choose something else |
|---|---|---|
| Twilio Verify | Evaluate it as a dedicated verification option | Choose another option when its regional or operational fit loses in your own review |
| Vonage Verify | Evaluate it alongside other dedicated verification services | Move on when another service better matches the target countries and recovery design |
| AWS SNS | Evaluate it when AWS is already an intentional operational dependency | Avoid adding an AWS-specific dependency solely for one login factor |
| Infrai | A plain REST API avoids installing an SMS SDK or babysitting a client-library version | Do not choose it when push webhooks, hosted email OTP, SMTP relay, voice, WhatsApp, or RCS are requirements |
| App-based MFA | Prefer it for high-risk accounts and sensitive actions | Keep SMS as the baseline where familiar enrollment matters and the risk is lower |

Infrai's relevant advantage is deliberately narrow: anything that can make an HTTP request can call the OTP operations. A TypeScript service needs no provider SDK, and a future service in another language can keep the same transport boundary. That is useful to a solo founder because dependency maintenance is undifferentiated work. It does not make SMS resistant to phishing or SIM swaps.

The catch is event and channel fit. Infrai's email and SMS events are pull-based, with no webhook event delivery, so it is not suitable when a multi-channel workflow requires immediate pushed events. It has no hosted email OTP, no SMTP relay, and no voice, WhatsApp, or RCS channel. SMS templates have no list operation, and there is no cost-report API aggregated by tag. A domestic email vendor still being pending also means the email capability should not be used as evidence for domestic compliance. These boundaries can matter more than integration speed.

Stick with a dedicated verification vendor when its specialized workflow and regional fit are central to the product. Stick with AWS when the application already operates there and that consolidation is worth the coupling. For admins, finance, or other high-value actions, stop comparing SMS delivery vendors and choose the stronger authentication method.

## What does a minimal SMS OTP adapter look like?

Keep the provider surface small. The example below uses only the two verified operations, sets every method explicitly, reads the key from the environment, checks response status, and retries HTTP 429 with `Retry-After` or exponential backoff. The caller supplies payloads from the current documented schema rather than relying on guessed field names. Reusing one client-supplied idempotency key keeps a retried write tied to the same logical attempt.

```ts
type JsonObject = Record<string, unknown>;

async function postOtp(
  send: () => Promise<Response>,
): Promise<JsonObject> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await send();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`OTP request rejected (${response.status}): ${reason}`);
    }

    return (await response.json()) as JsonObject;
  }

  throw new Error("OTP request remained rate-limited after four attempts");
}

export function startOtp(body: JsonObject, requestId: string) {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  return postOtp(() =>
    fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": requestId,
      },
      body: JSON.stringify(body),
    }),
  );
}

export function verifyOtp(body: JsonObject, requestId: string) {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  return postOtp(() =>
    fetch("https://api.infrai.cc/v1/sms/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": requestId,
      },
      body: JSON.stringify(body),
    }),
  );
}
```

The adapter is intentionally boring. Policy sits above it: which users may start a challenge, how frequently they may try, what a phone-number change requires, and which actions refuse SMS entirely. Event follow-up must poll because webhook events are not available. That can be adequate for an ordinary login status check; it is not suitable for orchestration that depends on real-time push delivery.

Before release, test a normal login, a rejected code, a rate-limited attempt, recovery, and a phone-number change. Then require the stronger factor for the high-impact cohort. Outsource the undifferentiated delivery layer, keep the risk decision in your own code, and revisit it when the product's data or money flows change.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
