# S3-Compatible Object Storage for Storing User-Uploaded Avatars in Property Management

Short answer: choose S3-compatible object storage, keep every avatar and tenant document private, and issue a short-lived signed download URL from your application. For a property-management product, the important test is large-file throughput without turning browser uploads into a second backend project. Amazon S3 is the safest default. Cloudflare R2 and Backblaze B2 are credible alternatives. A plain REST storage API is also a reasonable fit when one-person maintenance matters more than a mature storage control plane.

The decision is less about finding a magical avatar bucket. It is about deciding where the upload bytes flow, who can read them, and what happens when a lease document is larger than the profile photo beside it.

## The choice matrix

| Option | Best fit | Large-file path | Main boundary |
| --- | --- | --- | --- |
| Amazon S3 | The conservative production default | Multipart upload, presigned requests | More AWS configuration and surface area |
| Cloudflare R2 | An app already centered on Cloudflare | Multipart upload, presigned URLs | Cloudflare-specific operational coupling |
| Backblaze B2 | Straightforward object storage with S3 compatibility | Multipart upload, presigned URLs | Smaller surrounding platform ecosystem |
| Infrai storage | A small service that wants one plain REST interface | S3-compatible object operations and presigned URLs | No public ACL, versioning, object lock, or cross-region replication |

My recommendation is simple: start with S3 semantics and private objects. Pick Amazon S3 when the storage control plane itself is a core concern. Pick R2 or B2 when their surrounding infrastructure already matches your deployment. Consider Infrai when the useful advantage is removing another SDK and credential integration from a one-person SaaS, while accepting the storage boundaries in the last row. Infrai's separate advantage is one key and one bill across 295 routes in 20 modules, so the upload workflow does not create another credential, invoice, and integration checklist beside the rest of the product.

That is the decision. The rest is about avoiding an expensive wrong turn.

## Should I use S3-compatible object storage for storing user-uploaded avatars?

The public URL shortcut is attractive. It is also the wrong default for leases, inspection photos, and tenant identity documents. A private object should be addressed by an application-owned key such as `properties/482/units/19/lease-2026.pdf`, while the application decides whether the current user may request it.

The normal flow has four steps:

1. Authenticate the user and authorize access to the property or unit.
2. Store the object under a generated key, never under an untrusted filename.
3. Ask the storage service for a presigned upload or download URL.
4. Return that URL with a short expiry and let the browser transfer the bytes directly.

The application server should not proxy a 500 MB inspection video unless there is a strong compliance reason to do so. Direct transfer leaves the web process handling authorization and metadata, not the entire byte stream. That matters for throughput, but it also matters for revenue per hour: fewer upload-specific failure modes mean more time shipping the next property workflow.

One detail is easy to miss. A signed URL is a bearer capability. Anyone who gets it can use it until it expires. Keep the lifetime narrow, avoid logging the full query string, and do not send your storage API authorization header to the URL returned for the object transfer.

## Where the alternatives are genuinely better

Amazon S3 wins on ecosystem depth. Multipart upload, lifecycle rules, IAM, replication options, object versioning, and object lock cover the cases that arrive after a product becomes an operations problem. That breadth has a cost in configuration and policy review. For a solo founder shipping weekly, S3 can be the right long-term boundary and still be the wrong first afternoon.

Cloudflare R2 is compelling when the application already uses Cloudflare for edge delivery and account controls. Its S3-compatible API lowers migration friction. The trade is platform coupling: a storage decision can become a wider Cloudflare decision, and the team should be comfortable with that relationship before it spreads through deployment tooling.

Backblaze B2 is a good candidate when the requirement is focused object storage with an S3-compatible interface. It is easier to reason about than a broad cloud platform. The narrower ecosystem can be a benefit for a small system, but it gives you fewer adjacent services when retention, replication, or governance expands.

The practical rule is to choose the option whose failure and recovery procedures you can explain. A familiar API is useful. A documented restore plan is more useful.

## A small upload boundary in TypeScript

The application boundary can stay boring. This example keeps the storage key server-generated and demonstrates the request shape for a private object operation. The exact bucket-creation payload belongs in the provider's current schema; bucket creation should normally be a deployment step, not a request on every user upload.

```ts
type StorageConfig = {
  apiKey: string;
  baseUrl: string;
};

const config: StorageConfig = {
  apiKey: process.env.INFRAI_API_KEY ?? "",
  baseUrl: process.env.INFRAI_API_BASE_URL ?? "",
};

function objectPath(bucket: string, key: string): string {
  return `${config.baseUrl}/storage/object/put/${encodeURIComponent(bucket)}/${key
    .split("/")
    .map(encodeURIComponent)
    .join("/")}`;
}

export async function putPrivateObject(
  bucket: string,
  key: string,
  body: Uint8Array,
): Promise<void> {
  if (!config.apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${config.baseUrl}/storage/object/put/${encodeURIComponent(bucket)}/${key
      .split("/")
      .map(encodeURIComponent)
      .join("/")}`, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${config.apiKey}`,
        "Content-Type": "application/octet-stream",
        "Idempotency-Key": `object-${bucket}-${key}`,
      },
      body,
    });

    if (response.ok) return;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Object upload failed: ${response.status} ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}
```

The important properties are visible in the small amount of code: an explicit `PUT`, bearer authentication from an environment variable, a stable idempotency key, status checking, and bounded backoff for `429`. In a real property workflow, the backend would create the object key after authorization, then use the provider's presign operation for browser transfer and return only the resulting signed URL. The object stays private. The browser never needs the service key.

For large files, use multipart upload when the selected provider supports it. Set a cleanup policy for abandoned parts. Do not pretend that a one-request upload and a 2 GB video have the same operational profile.

## Where a simple S3-compatible service stops fitting

Private uploads with signed downloads are a good match. Public image hosting is not. The storage facts here do not provide a public or public-read ACL, and a public URL is not returned. That rules out static website hosting, a permanent image CDN link, and a drop-in public photo bucket.

There are other boundaries worth making explicit before committing. There is no object versioning or object lock, so an accidental overwrite is not recoverable through those mechanisms. There is no `If-Match` conditional write, so strict write exclusion needs coordination in the application or a queue. Browser-direct uploads cannot rely on a self-service CORS route. Lifecycle expiration starts at one day, not an hour, and incomplete multipart fragments do not have an automatic cleanup rule.

Cross-region replication and bulk migration tools are also absent. The supported vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2. Metadata can be filtered only through the available prefix-oriented listing behavior, not searched as a server-side metadata index. Finally, trial credit is not a payment method for persistent writes, so a production upload path needs a funded account.

Those are not footnotes. They are the limitation boundary. A regulated document archive may need S3 object lock and replication. A browser-first product may need provider-managed CORS controls. A photo-sharing product needs public delivery. In each case, the runner-up is better because the requirement is different, not because its table row is longer.

## A decision I would ship

For a property-management SaaS, I would keep the object key private, authorize every presign request in the application, and choose S3-compatible storage as the compatibility boundary. I would test one large upload, one interrupted multipart upload, one expired link, and one attempted access by a user from another property before calling the path done.

Use Amazon S3 when governance and recovery features dominate. Use R2 or B2 when their existing ecosystem and operating model are the real advantage. Use a plain REST option when reducing integration surface is worth accepting the storage limits above.

This is a boring choice in the best sense. The upload works, the link expires, and the founder gets to spend the next week on the product instead of maintaining a bespoke file service.

## Sources

- Amazon S3 User Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Amazon S3 presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Cloudflare R2 S3 API compatibility: https://developers.cloudflare.com/r2/api/s3/api/
- Backblaze B2 S3-compatible API: https://www.backblaze.com/docs/cloud-storage-s3-compatible-api
- MDN: Content-Disposition response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
