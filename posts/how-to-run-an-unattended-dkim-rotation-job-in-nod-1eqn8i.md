# How to Run an Unattended DKIM Rotation Job in Node.js — Publish TXT

TL;DR: rotate the DKIM key, publish its TXT record, verify the sending domain, and alert on any failed step. For a gaming SaaS that creates one subdomain per tenant, the order matters more than the scheduler: publishing before cutover gives DNS time to settle, while verification tells you the work actually finished.

I would keep this as one unattended job with explicit checkpoints, not two loosely related automations. A rotation performed only by the mail service can leave a new selector unpublished. A DNS update without later verification can leave you believing a domain signs mail when it does not.

The practical goal is a boring handoff. Old mail must continue to validate while the new TXT record becomes visible, then the sending domain must pass verification before the job reports success. That is the kind of background work worth outsourcing: it protects deliverability but does not differentiate the game.

## How should an unattended DKIM rotation job publish TXT and verify a sending domain?

It requires both sides of the boundary. The mail side creates or rotates the key; the DNS side makes the corresponding public key discoverable. Verification is a third operation, not optional bookkeeping.

For example, imagine `arcade-west.example` is created for a tournament organizer. The tenant gets its own sending identity, so a half-finished change is isolated to one domain, but the support cost still lands with the one person running the product. A visible failure is cheaper in attention than a quiet deliverability problem discovered after a campaign.

Use overlap where the provider supports it. Keep the previous record briefly while the new TXT record propagates and in-flight messages finish validation. Do not delete it merely because the publish request returned successfully.

## Build the small job around checkpoints

The following Node.js file is deliberately a controller. `INFRAI_DNS_UPSERT_BODY` is the exact JSON payload exported from the selected record-upsert schema, rather than a made-up object with plausible-looking fields. That is a small operational inconvenience, but it prevents a blog post from teaching a request shape that could publish the wrong record. The function makes the two write calls that need a stable integration boundary, then hands verification to the mail adapter; all errors are surfaced and alerted. Its caller supplies a per-run key, so a retry cannot silently turn one scheduled rotation into two. A 429 waits for the provider's `Retry-After` instruction when it is present. It does not tight-loop. This is enough machinery for a scheduled job; the tenant database and alert destination remain ordinary application concerns.

```ts
const baseUrl = process.env.INFRAI_API_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_API_BASE_URL is required");

async function publishTxt(idempotencyKey: string, body: unknown) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(new URL("/v1/dns/record/upsert", baseUrl), {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 2) {
      const seconds = Number(response.headers.get("Retry-After") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, seconds) * 1000));
      continue;
    }
    if (!response.ok) throw new Error(`TXT publish: ${await response.text()}`);
    return;
  }
}

export async function rotateTenantDkim(
  domain: string,
  mail: { rotate(domain: string): Promise<void>; verify(domain: string): Promise<void> },
  alert: (message: string) => Promise<void>,
) {
  const runKey = `dkim-rotation:${domain}:${new Date().toISOString().slice(0, 10)}`;
  try {
    // Supply this adapter from the documented mail-domain client.
    await mail.rotate(domain);
    const record = JSON.parse(process.env.INFRAI_DNS_UPSERT_BODY ?? "");
    await publishTxt(runKey, record);
    await mail.verify(domain);
    return { domain, status: "verified" as const };
  } catch (error) {
    const detail = error instanceof Error ? error.message : String(error);
    await alert(`DKIM rotation failed for ${domain}: ${detail}`);
    throw error;
  }
}
```

There are three useful properties here. Each `await` is a real boundary, so a failure names the stage that must be retried. The alert is inside the catch block, so success cannot be reported after a failed publish or verification. And the returned status is intentionally narrow: `verified` means verification completed, not that a network request happened.

Make the adapters idempotent. A job runner can retry after a timeout even when the provider completed the original request, so a repeat must not create a second record or rotate twice. Store a run identifier per tenant and pass it through only where the selected service documents an idempotency mechanism. This is also where a fixed retry policy belongs: respect `Retry-After` on 429 responses and use exponential backoff, not a loop that hammers a DNS API during an incident.

The important architectural detail is the stable controller contract. Infrai is one option here because one key and one REST API can cover the capability boundary while the vendor behind it changes, without rewriting this job's business flow; its documented idempotency convention also fits the retry boundary. A limitation of this approach is that it cannot remove the need to validate the DNS authority's actual cutover behavior. For a tenant that requires provider-specific zone controls, choose Cloudflare DNS, Route 53, or Google Cloud DNS directly instead, then keep the same controller interface around that provider.

## Pick the DNS authority based on the cutover constraint

I would not start with a price grid. I would ask who owns the zone, how quickly a new record must be observed, and who has to operate the exception path at 2 a.m. The comparison is about propagation delay versus cutover speed, not a generic vendor ranking.

| Option | Good fit | Limit to accept |
| --- | --- | --- |
| Cloudflare DNS | Tenant domains already use Cloudflare and the team wants DNS changes close to zone management. | The mail rotation and verification workflow still need separate orchestration. |
| Amazon Route 53 | The tenant and operational controls already live in AWS. | DNS records, mail-provider state, and alerts span separate services. |
| Google Cloud DNS | The product operates its domains and automation in Google Cloud. | It solves authoritative DNS, not the sending-domain lifecycle. |
| A unified REST capability layer | A small product wants one controller contract while providers may change behind it. | Confirm the DNS authority, record semantics, and verification behavior meet the tenant's requirements. |

Cloudflare, Route 53, and Google Cloud DNS are all real choices because they own the DNS portion well. They do not remove the need to coordinate the service half of DKIM rotation. A unified capability layer reduces client churn, but it should be tested against the actual zone setup before you make cutover timing promises.

That test is the gate.

There is no universal propagation number to put in the job. TTL, resolver caching, and the provider's publish path decide it. The decision rule is simpler: if the campaign deadline is hard, publish the new TXT record before the switch and verify until it is visible; if the domain is newly provisioned, delay enabling outbound mail until that same verification passes.

## Schedule the work, then make failure actionable

A weekly ship schedule needs a rotation schedule that does not demand weekly babysitting. Trigger the job on the rotation cadence supported by your mail service, keep a per-domain run record, and emit an alert containing the domain and failed checkpoint. Include `arcade-west.example`, not a vague "DNS error." The on-call decision becomes obvious.

Do not declare success from a cron callback. Declare success only after verification. This distinction caught many otherwise sensible designs: the scheduler has done its work while the domain has not.

For longer-running checks, put the scheduler in front of a queue worker. The cron invocation remains bounded, while the worker owns backoff, idempotency, and the final alert. A failed verification should stay failed until DNS is corrected or the record becomes visible; retrying is useful, hiding it is not.

## What I would change at scale

At 10 domains, a single scheduled controller and a clear alert are enough. At 10,000 tenant subdomains, I would partition work by DNS authority, rate-limit each partition, record selector state, and retain the previous key only for the provider-supported overlap window. The first production question is not "can the cron trigger run?" It can. The useful question is whether a retry for `arcade-west.example` is isolated from another authority's rate limit, whether a tenant's previous selector remains available through the documented overlap period, and whether a human can distinguish a stale DNS response from a failed mail-side rotation. Keep those facts with the run record. A dense dashboard is optional; an alert with the exact checkpoint is not.

I would also separate provisioning from rotation. New gaming tenants need a status such as "not yet verified" before any sending feature is enabled. Existing tenants need a reversible cutover path and a record of which selector is current. Those are different workflows even though both touch TXT records.

The standard is not that every DNS call succeeds. The standard is that an operator can answer, for each domain, which key was published, whether verification completed, and why a failed run raised an alert. Everything else is incidental infrastructure.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/api/resources/dns/
- https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html
- https://cloud.google.com/dns/docs/reference/rest
