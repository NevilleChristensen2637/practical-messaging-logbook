# US-EU SMS OTP Login Flow: 3 Evidence Gates for Secure Replay Protection

SMS delivery can be outsourced, but the evidence connecting a verified login to a support routing decision needs an owner. For a one-person automotive service SaaS, keep that evidence in the application and choose the verification boundary according to who can enforce attempts and consume a challenge atomically.

| Choice | Application evidence | Main trade-off |
| --- | --- | --- |
| Outsourced verification with application admission | Challenge reference, admission decisions, verification result, consumption event, routing policy version | Less authentication machinery to maintain; requires a clear concurrency and retry contract |
| Application-owned verification | The same decision record, plus responsibility for secret generation, storage, comparison, and consumption | More control over evidence; more security work competing with weekly feature delivery |

Short answer: favor outsourced verification when its documented behavior supports the evidence you need. Keep three application gates: admit the attempt, consume successful verification once, and authorize queue routing separately. Rate limits must survive replacement messages. Bound the retry budget and consume a successful challenge in a transaction. For US and EU customers, treat this as an authentication design; jurisdiction-specific compliance still needs a defined scope.

The concrete job is a contact form about an automotive service update. A customer verifies a phone number and asks about a repair status. The application must decide which support queue receives the request. Possession of the phone number is one input. Access to a particular service record is another.

## How should a secure SMS OTP login flow authorize support routing?

Use three gates because they answer different questions. Admission asks whether another send or verification attempt is allowed. Consumption asks whether this challenge has already produced a successful result. Authorization asks whether this account may access the service record and submit to the selected queue. Keep those decisions distinguishable in the audit record.

A valid code does not establish ownership of every repair order associated with a number typed into a form. Bind the challenge to the login transaction, normalized destination, and authentication purpose. After verification, resolve the account through the application's established account binding. Then check access to the requested service record before deriving a support queue from trusted application data. A queue identifier supplied by the browser is a request, not authority.

Keep that distinction.

For an illustrative policy, a challenge expires after 5 minutes, replacement messages have a cooldown of 60 seconds, and verification permits 5 failed guesses before that challenge is locked. These are proposed settings, not measured results or universal compliance thresholds. NIST's cited guidance requires at least six decimal digits for an out-of-band authentication secret, limits completion to ten minutes, requires rate limiting for short secrets, and requires accepting a secret only once during its validity period. It also treats authentication over the public switched telephone network as restricted. A five-minute timeout alone doesn't establish conformity with that guidance.

An evidence record should explain the decision without preserving the secret: an opaque challenge reference, account reference when established, purpose, creation and expiry times, attempt outcome, applicable policy version, and consumption event. Avoid copying the code or customer's message into authentication logs. For routing, retain the authorization outcome and queue policy version. Decide retention and access controls from the actual review requirements.

## Criterion one: can the record explain retries and lockout?

Retries make the policy harder. A customer requests another message because the first has not arrived. An attacker requests another message to reset the guess budget. The application needs to distinguish a delivery retry from a new authentication allowance even if both use the same button.

Keep the failure budget attached to the account or destination and authentication purpose across challenge replacement. Invalidate an older challenge when replacement is your chosen policy, but do not let replacement erase accumulated failures. Apply separate admission checks to message sends and code guesses. A send cooldown does not limit guesses against a code already issued. Destination limits alone also leave a gap when a client spreads requests across many numbers, so include a broader abuse budget with a documented scope.

Lockouts need boundaries. A challenge lock prevents further guesses against that challenge. An account restriction affects legitimate access more broadly and can itself become a denial-of-service tool. Prefer an explicit recovery path and stronger account checks for escalation over an indefinite account lock triggered solely by anonymous requests. Return a generic response before authentication so the form does not become an account lookup interface.

The useful audit question is precise: which counter denied this request, under which policy, and when may a legitimate customer try again? Record that internally. Do not expose account-existence implications in the public response.

For US and EU operations, define where counters are authoritative. Independent regional counters can each admit attempts that a shared policy intended to reject. If challenges belong to a home region, enforce that ownership on every verification request. If counters are shared, specify the consistency needed for admission. Regional storage placement, retention, and legal obligations require a separate review of the deployment and data flows; an OTP audit trail cannot answer them by itself.

## Criterion two: can success happen only once?

Checking the code and later marking it used creates a replay window. Two requests may both observe an unused challenge and continue. Expiry does not close that window. Neither does a browser disabling the submit button.

Consume it atomically.

The success transition must check expiry, lock state, purpose, and unused status at the same boundary that records consumption. When verification is outsourced, verify the service's concurrency and one-time acceptance contract. The application still needs to ensure that one verified result creates at most one login completion. A retry after an uncertain network result must recover a recorded outcome or require a fresh authentication step; it must not manufacture a second success.

Keep queue submission separate from code consumption. Customers can send another legitimate support request after signing in. Deduplicate retries of the same submission with an application-issued request reference bound to the authenticated account and submitted content. Reusing that reference with different content should fail. This lets the application retry routing without accepting an old code again.

Commit the authorized support request and its pending delivery event together. A worker can retry delivery while tracking the downstream acknowledgment. Claim delivery only when the receiving boundary confirms it. During support review, “verified,” “authorized,” and “delivered” describe different events.

## A focused implementation contract for the three gates

This TypeScript example describes an application boundary, not a complete authentication server. Its session comes from trusted server state after authentication. The store implementation must enforce the transaction contract; an interface provides no locking or durability.

```ts
type VerifiedSession = {
  accountId: string;
  authenticationEventId: string;
};

type SupportRequest = {
  submissionId: string;
  serviceRecordId: string;
  message: string;
};

type RoutingTransaction = {
  resolveAuthorizedQueue(
    accountId: string,
    serviceRecordId: string
  ): Promise<string | null>;

  // Atomically deduplicate by account and submission ID.
  // Reject different content; return the saved result for a retry.
  recordRequestAndPendingDelivery(input: {
    accountId: string;
    authenticationEventId: string;
    submission: SupportRequest;
    queueId: string;
    routingPolicyVersion: string;
  }): Promise<{ requestId: string }>;
};

type RoutingStore = {
  // Commit the request and pending delivery together, or roll both back.
  transaction<T>(
    work: (tx: RoutingTransaction) => Promise<T>
  ): Promise<T>;
};

async function routeSupportRequest(
  store: RoutingStore,
  session: VerifiedSession,
  submission: SupportRequest,
  routingPolicyVersion: string
): Promise<{ requestId: string }> {
  return store.transaction(async (tx) => {
    const queueId = await tx.resolveAuthorizedQueue(
      session.accountId,
      submission.serviceRecordId
    );

    if (queueId === null) {
      throw new Error("Support request cannot be authorized");
    }

    return tx.recordRequestAndPendingDelivery({
      accountId: session.accountId,
      authenticationEventId: session.authenticationEventId,
      submission,
      queueId,
      routingPolicyVersion
    });
  });
}
```

Before this function runs, authentication must have admitted verification and consumed the successful challenge. Do not construct `VerifiedSession` from browser-supplied account identifiers or an untrusted verification flag. Validate and bound contact-form input before storage. Protect the resulting session and request boundary too; one-time code acceptance does not protect a stolen session.

Test the contracts that carry the risk. Submit the same valid code concurrently and require one consumption event. Replace a challenge after several failures and verify that the failure budget survives. Retry one support submission and require one pending delivery record. Change its content under the same reference and require rejection. Finally, authenticate an account without access to the requested service record and verify that routing is denied.

Ship weekly, with reviewable policy changes. Version admission and routing rules, deploy with explicit rollback behavior, and monitor denial reasons, consumption conflicts, and pending delivery age without recording secrets. A rollback must preserve consumed challenges and accumulated failures. Otherwise, a routine release can reopen decisions the evidence says are closed.

## When owning verification is the better trade

Application-owned verification becomes the stronger choice when an external contract cannot answer the required questions about attempt accounting, concurrency, or evidence access. It also fits when the application already has a maintained authentication component supplying these guarantees. The trade is ongoing responsibility for security-sensitive behavior, including generation and storage of secrets, abuse handling, and recovery.

Use revenue per engineering hour as a planning lens. Estimate recurring maintenance alongside weekly feature delivery. Outsource the undifferentiated delivery work where that reduces operational responsibility, while retaining enough evidence to explain every support handoff. The decision turns on demonstrable enforcement and a reviewable record, not the message's unit price.

Neither choice makes SMS possession sufficient for unrestricted access to customer records. Match authentication to the risk and applicable requirements. Keep authorization explicit at the support boundary.

## References

The authentication requirements discussed here come from the NIST guidance listed below and apply within its stated scope. The illustrative settings in this note are application policy choices.

The DKIM specification concerns email domain signatures. It does not establish SMS possession, support-record authorization, or compliance for this flow. It is relevant only if a separate email path is being reviewed.

## Sources

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
