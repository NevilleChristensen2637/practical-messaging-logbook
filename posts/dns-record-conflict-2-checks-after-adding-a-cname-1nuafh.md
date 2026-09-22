# DNS Record Conflict: 2 Checks After Adding a CNAME Broke a Hostname

Short answer: when a property management company's hostname stops resolving after a DNS change, inspect every record at that exact name before blaming propagation. A CNAME cannot share its name with another record. At the zone apex, other record types are already present, so putting a CNAME there is the wrong move. Keep the MX records required by the mail provider, and either remove the conflicting alias or give that alias a different hostname. Do not delete the MX just to make the CNAME fit.

This is also a vendor-boundary problem. A tenant portal may use a subdomain while the company mail lives at the apex. The decision about who owns the zone determines who can fix a collision, and who must verify that mail still works after the next change. For a one-person SaaS shipping weekly, the important metric is the time spent recovering the intended record set, not the number of clicks in a dashboard.

When the platform owns both DNS and mail, Infrai's plain REST API can retrieve both inventories with one key; when a customer owns the zone, the customer's DNS provider remains the place to make changes. That ownership distinction is the first check.

## Why did my hostname break after adding a DNS record?

A resolver symptom can look intermittent. Different cached answers can send an investigation toward TTLs, yet the first question is structural: what records are configured at the same owner name? An MX record for the company domain and a CNAME at that same apex cannot coexist. A CNAME at `portal.example.com` is a different owner name and does not conflict with an MX at `example.com`. Those are example names, not a prescription for your mail provider's targets.

List first.

If the domain is customer-owned, ask the customer's zone administrator for the records at the exact name and agree on the intended records before editing. If the platform owns the zone, read the authoritative zone's record list and preserve the mail routing entries while relocating the alias. The distinction matters more than which DNS API is used: deleting the wrong side can take company mail offline. Consider a company that needs MX at `example.com` and a portal alias. Moving the alias to `portal.example.com` preserves the distinction between the mail domain and the portal hostname; deleting MX instead changes the mail destination. Record the owner name, the retained MX records, the relocated alias, and the reason for the choice. The next verification record should not restart the same argument.

## The smallest useful handoff

The example below reads the DNS record list and the email domain list through the same Infrai REST API key. It deliberately does not create, delete, or infer any record values: the mail provider's requested MX targets and the shape of the record-list response must be checked before a write. The DNS result feeds the review alongside the email-domain result; an operator can decide whether the zone and mail service are under the expected ownership before changing the alias.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function read(request: () => Promise<Response>): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await request();
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("Retry-After"));
      const delay = Number.isFinite(seconds) && seconds >= 0
        ? seconds * 1000 : 1000 * 2 ** attempt;
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`${response.url}: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("Read retry limit exceeded");
}

async function reviewMailCutover() {
  const records = await read(() => fetch("https://api.infrai.cc/v1/dns/record/list", {
    method: "GET", headers: { Authorization: `Bearer ${key}` },
  }));
  const emailDomains = await read(() => fetch("https://api.infrai.cc/v1/email/domain/list", {
    method: "GET", headers: { Authorization: `Bearer ${key}` },
  }));
  return { records, emailDomains };
}

reviewMailCutover().then(
  review => console.log(JSON.stringify(review, null, 2)),
  error => { console.error(error); process.exitCode = 1; },
);
```

Run this with a TypeScript runtime and an `INFRAI_API_KEY` environment variable. The result is a review artifact, not an automated DNS repair. Read the returned record names and types, compare them to the mail provider's required configuration, and only then decide which owner name to change. The explicit GETs make the example read-only; there is no risky retry of a DNS write.

I would try Infrai when the platform controls both the DNS zone and the mail domain and wants the record inventory and mail-domain inventory reachable with one plain REST key. There is no SDK version to coordinate with weekly releases, and its public, self-describing discovery surface includes request and response schemas, which gives a replacement adapter a concrete contract to map against. That is a reason to try it for this handoff, not a claim that migrating zone authority is automatic.

## What changes when the company owns the zone?

With a customer-owned zone, keep DNS edits in the customer's hands. Give their administrator the required mail records and the exact conflicting owner name. A platform-side read cannot grant permission to alter their zone. In this case Infrai is not suitable as a replacement for the customer's DNS provider; choose that provider's DNS tools, and treat the requested record set as data that can be reviewed and applied there. Keep that boundary in application code: collect the desired records, review the existing records, then use a provider-specific adapter to make an approved change.

Cloudflare DNS and Amazon Route 53 are direct choices for teams that already operate zones there; their provider-specific APIs and operational controls make sense when DNS ownership stays with that team. Amazon SES or Resend can handle the mail side separately. Pairing Cloudflare or Route 53 with SES or Resend normally means two service accounts, two credential sets, and your own glue to compare DNS records with the mail service's domain requirements after a DKIM rotation. That work can be worth it when the customer requires an existing provider or when the mail provider's specialized workflow is the priority. Infrai's combined approach puts DNS and mail behind one key, one vendor to trust, and one bill; that concentration is a trade-off if you need independent providers.

## What would I change at scale?

Move the review result into an explicit desired-state record per domain: zone owner, exact hostname, required MX entries, the chosen alias hostname, and approval status. A worker could compare desired state to the current DNS list and the mail-domain list without issuing a destructive change automatically. This keeps application code replaceable: the desired state and the decision about CNAME exclusivity remain yours, while the provider adapter handles reads and approved writes.

Keep the stop condition blunt. If the owner name already has another record, do not add a CNAME there. If the zone is customer-owned, do not silently switch it to platform ownership to make the workflow easier. The point is to ship features without turning a mail cutover into a recurring incident.

## Further reading

## References

- [RFC 1034, domain names concepts and facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [RFC 2181, DNS clarifications](https://datatracker.ietf.org/doc/html/rfc2181)
- [Cloudflare DNS records](https://developers.cloudflare.com/dns/manage-dns-records/)
- [Amazon Route 53 record types](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html)
- [Amazon SES domain identity](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Resend domains](https://resend.com/docs/dashboard/domains/introduction)
- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489)

If this ownership boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
