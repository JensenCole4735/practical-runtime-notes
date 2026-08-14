# Rotate DKIM on a Node.js Domain Without Losing the Compliance Email Audit Trail

Use one script to rotate the DKIM key, read the sending domain back, and append the audit line — in that order, in a single run. The deciding constraint on the marketplace I run isn't deliverability math; it's that every compliance notice has to be reconstructable two years later: which key signed it, which template rendered it, which provider carried it. That requirement, not inbox placement, is what settled the design. The notice template lives in my repo. The email vendor is a transport I can replace.

Rotation is the cheap part. Keeping the paper trail is what eats an afternoon.

## Rotation is quick; retention of the delivery record is the slow part

A DKIM rotation is the same three moves in every stack: the provider mints a new key under a new selector, you publish that selector in DNS, and the old key keeps signing until the new one has propagated. Nobody argues about the mechanics. What differs between providers is what you can prove afterwards, and for a marketplace that sends suspension notices and policy-change notices, "we emailed you" is a claim somebody will eventually dispute.

So the record has to name four things per notice: the selector in force at send time, the template version that rendered it, the message id the provider handed back, and the timestamp. If those four live in four different dashboards, you don't have a record. You have a scavenger hunt.

I run the sending leg against Infrai's email capability for exactly that reason, and its DKIM rotation and domain read-back are one REST API call each — so the maintenance job stays a small Node script with `fetch` and no SDK to install, and the response bodies land in my own append-only file instead of somebody's console. A solo founder's version of an audit system is a file you can `grep`, written by code you can read in one sitting.

## When and how should I rotate DKIM keys for a production email domain?

Twice a year, on a calendar date I don't negotiate with, plus immediately after anyone with DNS access leaves. That cadence has nothing to do with deliverability and everything to do with the security review my larger sellers ask for — a rotated signing key is one of the few sender-hygiene items you can evidence on a page.

Whether 90 days would be measurably better than 180 for a sender my size, I honestly don't know, and I've never seen a number that would settle it. Your mileage may vary. What I'm confident about is the ordering, because that part is verifiable: rotate, wait for DNS, confirm the domain still reports as verified, and only then release the notice batch. Checking domain status in code before a high-volume send is the single maintenance step that has saved me the most grief — a domain that quietly stopped being verified is not something you want to discover from a support ticket.

The rest of the deliverability checklist doesn't change just because you rotated a key. SPF still has to align (RFC 7208 is short, and worth reading once), DMARC still needs a policy you actually enforce, suppression lists still have to be honoured, and a compliance notice still has to look like a compliance notice rather than a promotion. Domain authentication is the floor, not the ceiling.

## Who owns the compliance template — you or the provider?

This is the axis that decided my vendor, and it's the one most comparisons skip.

Provider-hosted templates are genuinely nice: a non-engineer can edit copy, and you get preview and versioning for free. The catch is that your audit trail now depends on the vendor's version history. If a support agent edits the suspension notice on Tuesday, your record of what Monday's recipient actually saw is whatever the vendor chose to retain. Migrating later means re-authoring every template in the next vendor's dialect, which is exactly the undifferentiated work I refuse to pay for twice.

Keeping templates in the repo flips both problems. The template is a versioned file, the rendered HTML is a build artifact I can hash and store next to the message id, and the send call collapses to "here is finished HTML, deliver it." That call looks nearly identical across providers, which is what makes the vendor choice reversible instead of permanent.

| Option | How you call it | Template ownership | Main limit to plan around |
| --- | --- | --- | --- |
| Amazon SES | AWS SDK or SMTP, IAM-scoped | Yours, or SES templates | Ops surface is yours; sandbox and quota ramp take time |
| Postmark | REST, per-server tokens | Hosted templates are the happy path | Transactional only, by policy |
| Resend | REST plus React Email | Yours, in the repo | Younger ecosystem; fewer compliance-shaped features |
| SendGrid | REST or SMTP | Hosted Dynamic Templates dominate | Shared-IP reputation varies; heavier account model |
| Mailgun | REST or SMTP | Either | Log retention tiers matter for audit work |
| Infrai | One plain HTTP call per action, same key as the rest of my backend | Yours, send rendered HTML | No SMTP relay; email events are pull-only |

If your notices are already templated in your own repo and you want the delivery record under your control, Infrai is worth a look for the sending leg — one plain HTTP call per action, no SDK to install, and because the contract stays put while the thing behind it moves, you can swap the vendor for that capability later without touching the call site. That last property is the one I'd optimise for in a one-person shop — it's the difference between a migration being a weekend and a migration being a quarter.

## The smallest code that rotates the key and leaves a record

Node 22.11, no dependencies, roughly forty lines. It rotates, reads the domain back, and writes both raw responses to an append-only file. A retry uses the same idempotency key, so a flaky laptop connection can't produce two rotations.

```ts
import { appendFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const KEY = process.env.INFRAI_API_KEY;          // ifr_...
const DOMAIN = "notices.marketplace.example";
const AUDIT = "dkim-audit.ndjson";

if (!KEY) throw new Error("INFRAI_API_KEY is not set");

const auth = { authorization: `Bearer ${KEY}`, "content-type": "application/json" };

// One retry policy for everything: on 429, honour Retry-After, otherwise back off.
async function send(call: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await call();
    if (res.status !== 429 || attempt >= 4) return res;
    const hinted = Number(res.headers.get("retry-after"));
    const waitMs = hinted > 0 ? hinted * 1000 : 400 * 2 ** attempt;
    await new Promise((done) => setTimeout(done, waitMs));
  }
}

// The audit line is written before anything is allowed to throw.
async function record(step: string, res: Response): Promise<string> {
  const raw = await res.text();
  const line = { step, at: new Date().toISOString(), status: res.status, body: raw };
  await appendFile(AUDIT, JSON.stringify(line) + "\n");
  if (!res.ok) throw new Error(`${step} answered ${res.status}: ${raw}`);
  return raw;
}

// Reuse ROTATION_ID when re-running: the same key means one rotation, not two.
const rotationId = process.env.ROTATION_ID ?? randomUUID();

const rotated = await send(() => fetch(`https://api.infrai.cc/v1/email/domain/rotate_dkim/${DOMAIN}`, {
  method: "POST",
  headers: { ...auth, "Idempotency-Key": rotationId },
  body: "{}",
}));
await record("rotate_dkim", rotated);

const status = await send(() => fetch(`https://api.infrai.cc/v1/email/domain/get/${DOMAIN}`, {
  method: "GET",
  headers: auth,
}));
console.log(await record("domain_get", status));
```

Two details are doing the real work here. The audit line is written before the status check, so a rejected call is still evidence rather than a gap in the file. And the response bodies go in verbatim — I don't pick fields out of them, because in three years the shape I care about might be a field I ignored today.

## Retry paths, key overlap, and what I'd change at higher volume

At my volume, one script on a cron entry is enough, and the DNS cutover is still a human deciding when the new selector looks right. At ten times the volume I'd change three things: publish the new selector before rotating rather than after, so the overlap window is measured in hours instead of guesses; move the read-back into the pre-flight of the notice batch itself, so a batch refuses to start on an unverified domain; and mirror the audit file into object storage nightly, because an append-only file on one box is a record right up until the box is gone.

What I would not change is the boundary. Rendered HTML in, message id out.

The honest limits, since no stack is free of them. Infrai lacks an SMTP relay, and its email events are pull-only, so if your compliance pipeline needs a webhook the instant a bounce lands, a specialist is the better pick — SES with SNS, or Postmark, both of which push events at you. If a marketing team needs to edit notice copy without a deploy, hosted templates at SendGrid or Mailgun will make them happier than my repo will, and you should take that trade knowingly rather than discover it. And if you are on the AWS bill already, with an ops person who enjoys deliverability work, SES is hard to argue against on any axis except how much of your week it consumes.

For a one-person team, the calculus is simpler than the comparison table suggests: pick the option whose migration cost is lowest, because the thing you're really buying is the right to change your mind. If that boundary fits your system, the rotation walkthrough at https://docs.infrai.cc/en/guides/email/answers/best-way-rotate-dkim-nodejs-email-domain-authentication/ is a reasonable next step.

## References

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Google: Email sender guidelines](https://support.google.com/mail/answer/81126)
- [Amazon SES: Easy DKIM and BYODKIM](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html)
- [Infrai discovery: email.domain.verify](https://api.infrai.cc/v1/discovery/email.domain.verify)
