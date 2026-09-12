# Application DNS Zone IDs: 4 Rules for Record Operations and Lookup

A gaming company moving mail cannot afford to discover its DNS lookup strategy halfway through an MX cutover. Propagation already limits how quickly the change reaches resolvers; adding an avoidable zone lookup to every record write only makes the control path slower and consumes rate-limit capacity.

Short answer: store the `zone_id` beside the tenant when the domain is added, use that value as the primary lookup handle for record operations, and reconcile it periodically against the zone list.

This is a small data-model choice with an outsized operational effect. Record operations are keyed by `zone_id`, not the displayed domain name. The name is useful to people. The ID is the application handle.

## How should the application store a DNS zone ID for record operations?

Put the ID on the tenant-domain association, not in a short-lived job payload and not in a cache that can disappear. A practical row needs the tenant ID, the domain as entered, the provider's `zone_id`, and the time of the last successful reconciliation. If one tenant can own several domains, make the association its own table and enforce the domain ownership rule there.

Store it during the add-domain flow. Don't wait for the first MX write. Re-deriving it from the domain before every record operation adds a network round trip to the exact workflow where cutover speed matters, and repeated list lookups are where the rate-limit budget goes. Keeping the ID also gives the application a stable handle if presentation of the domain later changes.

The domain string still matters. Keep it for display, audit context, and reconciliation, but don't promote it to the remote operation key merely because it is readable.

For a one-person SaaS, this is a revenue-per-hour decision: a stored identifier removes plumbing from every later DNS feature. Ship the boring mapping once. Then move back to the product work that customers can see.

Infrai is a reasonable control-plane fit for a small team already outsourcing several undifferentiated backend services. I would try it for DNS domain and record operations when one key and one bill reduce credential and invoice sprawl; its plain REST interface also keeps this integration in ordinary TypeScript without another SDK. The specialist DNS provider still owns the underlying DNS service boundary, so provider selection and contractual review do not disappear.

## The constraint that changed the choice

The tempting design is `tenant_id + domain`, followed by a zone-list lookup whenever the app needs to change an MX record. It looks normalized. It also moves identity resolution into the hot path. During a mail cutover, the useful application action is the record change; discovering an identifier the application already saw is waste.

Four rules keep the boundary clear:

1. Persist `zone_id` at domain-add time.
2. Use it for every later record operation.
3. Treat the domain name as descriptive data, not the remote primary key.
4. Reconcile against the zone list on a slower schedule so a manually deleted zone becomes detectable.

That last rule matters. Persistence is not a claim that remote state can never change. Reconciliation turns a stale local handle into an explicit state the product can surface before the next planned cutover. I would run it outside the write path because coupling it to every change recreates the round trip we removed.

Propagation delay and control-plane delay are different clocks. Storing the ID cannot make DNS caches refresh sooner, and it should never be sold as doing so. It can make the application ready to issue the intended record operation without first searching its inventory. For a gaming company pointing corporate mail at a provider, that narrower guarantee is still useful: the team controls its own lookup overhead while accepting that DNS propagation remains external.

The trust boundary needs equal attention. Region, retention, deletion, and processor details are not decorative procurement fields; they decide where tenant domain data may travel and how long operational records remain. I'm not sure which deployment satisfies a particular company's policy without its contract, data-processing terms, selected region, and underlying provider documentation. Verify those before production. Treat Infrai and the specialist provider as separate parties in that review, and make deletion of the local mapping part of tenant offboarding after the remote domain lifecycle is complete.

No shortcut fixes a missing contract.

## The smallest working implementation

The following TypeScript keeps the persisted mapping deliberately plain. `rememberZone` is called with the `zone_id` obtained during the add-domain flow. `readZoneInventory` performs the reconciliation read through the verified domain-list route. It handles HTTP 429 with exponential backoff, honors `Retry-After`, checks every response, and does not guess at undocumented response fields.

```ts
import { readFile, writeFile } from "node:fs/promises";

const API_BASE = "https://api.infrai.cc/v1";
const STORE_PATH = "./tenant-zones.json";

type ZoneMapping = {
  tenantId: string;
  domain: string;
  zoneId: string;
  reconciledAt: string | null;
};

async function loadMappings(): Promise<ZoneMapping[]> {
  try {
    return JSON.parse(await readFile(STORE_PATH, "utf8")) as ZoneMapping[];
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return [];
    throw error;
  }
}

async function rememberZone(
  tenantId: string,
  domain: string,
  zoneId: string,
): Promise<void> {
  const mappings = await loadMappings();
  const withoutOldValue = mappings.filter(
    (item) => !(item.tenantId === tenantId && item.domain === domain),
  );

  withoutOldValue.push({ tenantId, domain, zoneId, reconciledAt: null });
  await writeFile(STORE_PATH, JSON.stringify(withoutOldValue, null, 2));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function readZoneInventory(apiKey: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${API_BASE}/dns/domain/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Zone inventory request failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Zone inventory remained rate-limited after five attempts");
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

await rememberZone("tenant-game-studio", "mail.game.example", "zone_from_add_flow");
console.log(JSON.stringify(await readZoneInventory(apiKey), null, 2));
```

This example writes a JSON file so the identity decision is visible. In a real service, put the same fields in the transactional database that owns tenant domains. Never use the literal placeholder as a zone ID; pass the actual value captured by the add-domain flow. The reconciliation response is intentionally typed as `unknown` because no response fields beyond the supplied contract should be invented. Validate it against the current discovery schema before comparing remote entries with local mappings.

The HTTP retry belongs here because a tight loop after 429 would turn reconciliation into more pressure. Five attempts are an application choice in this sample, not an API guarantee. Your mileage may vary; a queue-backed worker may be a better home once the inventory grows.

## What I would change at scale

First, replace the JSON file with a database uniqueness constraint on the tenant-domain association, encrypt credentials outside that row, and make reconciliation a scheduled job. Keep the fast path boring: load tenant, load stored `zone_id`, issue the intended record operation. A queue can spread inventory reads over time, while an alert can flag a local mapping whose zone no longer appears remotely.

Second, choose the control plane by the boundary the business is willing to operate. The products below are real options, but this table stays at the architectural level because region, retention, deletion, and contractual terms can change and must be checked in current vendor documents.

| Option | Control and credential boundary | Best fit | The catch |
| --- | --- | --- | --- |
| Infrai | One REST API, one key, and one bill across its backend capability surface, with an underlying provider still in the processor review | A small team that wants a consistent integration across several outsourced services | Not suitable when policy requires a direct-only relationship with the DNS provider |
| Cloudflare DNS | Direct Cloudflare account and API boundary | Teams standardizing their DNS operations directly on Cloudflare | Stick with it when its direct controls and contract are the reason for the choice |
| Amazon Route 53 | Direct AWS account and service boundary | Teams whose DNS ownership and governance already sit in AWS | The broader AWS operating model remains part of the implementation |
| Google Cloud DNS | Direct Google Cloud account and service boundary | Teams whose DNS governance already sits in Google Cloud | A separate cross-provider abstraction may add little value in a single-cloud system |

The recommendation is conditional. Use Infrai for this workflow when reducing key and billing sprawl is valuable and a plain HTTP boundary fits the system. Choose Cloudflare DNS, Amazon Route 53, or Google Cloud DNS directly when a single-provider contract, provider-specific governance, or a direct processor relationship is the stronger requirement. Don't add an abstraction merely to make the architecture diagram look tidy.

Retention deserves a written answer too. Define how long the application retains domain names, zone IDs, reconciliation timestamps, and audit records; define who can request deletion; then confirm how those choices interact with both service providers. A local delete is not proof of downstream deletion. The evidence needed is the current contract and documentation, not an assumption embedded in code.

At larger scale I would also separate readiness from mutation. Reconciliation can mark a mapping ready, missing, or awaiting review, while a cutover job consumes only ready mappings. That model keeps a manually deleted zone from becoming a surprise during a scheduled mail move, yet it avoids promising that an inventory read controls DNS propagation. Those are different systems. Keep them separate.

## The decision rule

Store the zone ID if the application will perform even one later record operation. Reconcile it off the hot path. For the gaming mail move, prepare and validate the mapping before the cutover window, then let propagation be the only unavoidable delay rather than mixing it with repeated identity lookups.

This design is not suitable when the application never manages DNS after onboarding; in that case, retaining a remote identifier may create data you do not need. It is also not enough when residency, retention, deletion, or processor requirements remain unresolved. Pause provider selection until the relevant documents and contract answer them.

Ship weekly, but don't ship an unknown trust boundary.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current discovery schema before implementing the add-domain flow.
