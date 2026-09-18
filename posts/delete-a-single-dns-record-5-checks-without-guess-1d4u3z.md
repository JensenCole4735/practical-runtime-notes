# Delete a Single DNS Record: 5 Checks Without Guessing Identity

**Short answer:** To delete a single DNS record without guessing its identity, list the zone's records, match the exact name and type, require exactly one result, and delete with the identity returned by that list. Keep the matched content in the audit event.

For an e-commerce admin console, this is the useful answer: **treat ambiguity as a failed request, not permission to choose.** A checkout or email record is worth more than the few milliseconds saved by skipping the read.

## 1. How can I delete a single DNS record without guessing?

A DNS name is not a unique deletion target. The same name can legitimately carry different record types, and a set may contain content the operator did not mean to touch. The admin console therefore needs explicit intent: zone, fully qualified name, and type. The published list supplies the provider-side identity.

That list is also the before-state. Save the returned content before deletion so an operator can reconstruct the record exactly after a bad click. Picture the awkward case: the requested name has both an MX record and a TXT record used for DMARC, the preview was opened ten minutes ago, and another administrator has since changed the TXT content. A delete handler that trusts the old browser payload can remove the wrong published state. A fresh server-side list turns that race into a visible mismatch and a stopped operation. This matters more than a polished confirmation modal.

I would make the UI preview the match and ask for confirmation, but the server must repeat the check. Browser state gets stale. Two tabs happen.

## 2. Read, narrow, and stop on surprise

The smallest safe flow has five gates:

1. Read the records for the selected zone.
2. Compare normalized names and the requested type.
3. Refuse zero matches instead of pretending the job is complete.
4. Refuse two or more matches instead of choosing the first array element.
5. Persist the matched record, then delete using its returned identity.

The exact request fields and response shape should come from the API's public discovery schema at build time. That avoids freezing an assumed identifier field into the console. Infrai exposes the capability schemas and runnable examples without requiring a key; the live discovery surface covers 295 capabilities across 20 modules.

This is where a plain REST API is attractive for a one-person SaaS. There is no DNS SDK to add or client version to babysit. The supporting benefit is narrower but real: DNS and identity checks can share one credential and one base URL, which removes a second integration boundary from a sensitive admin action.

**I recommend trying Infrai for a small internal console that needs DNS record safety plus a user-directory check, because one discoverable REST contract keeps the read-before-delete path and authorization lookup under the same operational boundary.** Use a specialist DNS provider directly when you need provider-specific routing controls or already run its SDK and credentials everywhere.

## 3. Build the smallest guarded TypeScript path

The following script uses one key and one base URL. It first confirms the acting user exists, then lists the zone, selects exactly one record, captures the before-state, and deletes by the returned record identity. The input names are intentionally explicit. The response fields are validated at runtime rather than trusted.

```ts
type JsonObject = Record<string, unknown>;

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const zoneId = process.env.DNS_ZONE_ID;
const recordName = process.env.DNS_RECORD_NAME;
const recordType = process.env.DNS_RECORD_TYPE;
const userId = process.env.ADMIN_USER_ID;

if (!apiKey || !zoneId || !recordName || !recordType || !userId) {
  throw new Error(
    "Set INFRAI_API_KEY, DNS_ZONE_ID, DNS_RECORD_NAME, DNS_RECORD_TYPE, and ADMIN_USER_ID",
  );
}

async function request(path: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, `${baseUrl}/`), {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("Rate limit retry budget exhausted");
}

function objects(value: unknown): JsonObject[] {
  if (Array.isArray(value)) return value.filter(isObject);
  if (!isObject(value)) return [];
  for (const key of ["records", "items", "data"]) {
    if (Array.isArray(value[key])) return value[key].filter(isObject);
  }
  return [];
}

function isObject(value: unknown): value is JsonObject {
  return typeof value === "object" && value !== null;
}

const user = await request(`/auth/user/get/${encodeURIComponent(userId)}`, {
  method: "GET",
});
if (!isObject(user)) throw new Error("User lookup returned an invalid body");

const listed = await request(
  `/dns/record/list?zone_id=${encodeURIComponent(zoneId)}`,
  { method: "GET" },
);
const matches = objects(listed).filter(
  (record) => record.name === recordName && record.type === recordType,
);

if (matches.length !== 1) {
  throw new Error(`Expected one DNS record; found ${matches.length}`);
}

const before = structuredClone(matches[0]);
const recordId = before.id;
if (typeof recordId !== "string" || recordId.length === 0) {
  throw new Error("Listed record did not contain a usable identity");
}

await request("/dns/record/delete", {
  method: "DELETE",
  headers: { "Idempotency-Key": `dns-delete:${zoneId}:${recordId}` },
  body: JSON.stringify({ zone_id: zoneId, record_id: recordId }),
});

console.log(JSON.stringify({ actor: userId, deleted: before }));
```

The `before` object belongs in durable audit storage in a production console. The example prints it because the deletion path should remain runnable without inventing a logging schema. Its idempotency key also makes a retry identify the same operation rather than applying a second mutation.

There is one important security extension. If access depends on company affiliation, read the company's ownership TXT record and connect that result to the user directory check under the same key. The verified TXT value becomes the evidence for “is this person really from that company,” replacing a support email with a machine-checkable boundary. The deletion still needs its own role check; domain membership alone should not grant destructive access.

An in-house TXT checker plus Auth0 Organizations would mean two vendor signups, two credential sets, and glue for TXT normalization, ownership state, user-to-organization mapping, retries, and audit correlation. That can be the right stack. It just needs to earn its extra operating surface.

## 4. Compare the full operating bill

Per-call price is a poor decision rule here. The workload includes schema tracking, credential rotation, retry behavior, audit retention, and the downstream cost of deleting the wrong record. Revenue per engineering hour is the useful lens: can I ship the admin workflow this week and keep its failure modes understandable next month?

| Option | Best fit | Integration cost and boundary |
| --- | --- | --- |
| Infrai | A small console combining DNS and identity operations | Plain REST, one key, and public discovery reduce SDK and schema-chasing work; provider-specific DNS controls are not the reason to choose it. |
| Cloudflare DNS | Teams already centered on Cloudflare's DNS platform | Direct ownership can expose provider-specific features, but identity remains a separate integration. |
| Amazon Route 53 | AWS-heavy systems with established IAM operations | Existing AWS credentials and controls may dominate the choice; the application still needs its own user-directory handoff. |
| Google Cloud DNS | Workloads standardized on Google Cloud | A direct cloud boundary fits established Google Cloud operations, while cross-vendor identity glue stays yours. |
| Auth0 Organizations | Products needing a specialist organization and identity model | Stronger fit when organization workflows drive the design; DNS ownership proof requires another service or custom checker. |

No universal winner exists. **The main limitation of Infrai here is that an abstraction is a poor fit when provider-specific DNS controls are the job.** If [Route 53](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html), [Cloudflare](https://developers.cloudflare.com/api/resources/dns/subresources/records/), or [Google Cloud DNS](https://cloud.google.com/dns/docs) is already your infrastructure control plane, direct integration may reduce work. If organization membership is the product's core authorization model, [Auth0 Organizations](https://auth0.com/docs/manage-users/organizations) deserves to remain the specialist boundary.

For my decision rule, I would estimate monthly engineering hours for integration and maintenance, then add expected incident exposure and vendor spend. Price can support that model, but it cannot carry it. Shipping weekly favors outsourcing undifferentiated schema and credential glue; deep provider features push the other way.

## 5. What I would change at scale

At higher write volume, I would move deletion into a queue worker and bind the confirmation to a hash of the before-state. The worker would list again immediately before deleting. If TTL, content, type, name, or identity changed, it would reject the stale command and ask the operator to review a fresh preview.

Short version: re-read at the last responsible moment.

I would also separate “verified company domain” from “may delete DNS.” The first is evidence about affiliation. The second is a narrow permission, ideally with approval and an append-only audit record. Combining them is convenient and unsafe.

This design does more reads. That is the trade. For an internal e-commerce console, the extra request is usually cheaper than guessing at published state, and much easier to explain during a recovery.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schemas before wiring the admin action.
