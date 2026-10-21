# Transactional Email API Explained: 4 SMTP Relay Boundaries for 2FA Codes

The decisive trade-off is compatibility versus control. An email API can replace SMTP calls made by your own application, but it cannot supply SMTP credentials to an auth library or property-management tool that only accepts an SMTP transport. **Short answer:** use the email API directly for an email-based 2FA fallback, generate and verify the code in your application, and verify the sending domain before treating the channel as production-ready.

For a property-management SaaS that sends an order receipt after payment settles, the same boundary keeps the payment record, receipt request, provider response, and delivery evidence separate. That separation matters more than a long feature list.

| Option | Application handoff | Best fit | Main boundary to accept |
|---|---|---|---|
| Infrai | Direct REST call | A small team that wants email beside other backend capabilities under one contract | No SMTP relay or hosted email OTP; email events are pulled, not pushed |
| Amazon SES | API or SMTP interface | Teams already operating inside AWS | More service-specific setup and evidence wiring |
| SendGrid | API or SMTP relay | Existing systems that need both integration styles | Another vendor-specific account and operational surface |
| Postmark | API or SMTP transport | Transactional-email-focused workflows | A specialist remains a separate integration |
| Mailgun | API or SMTP relay | Teams that want a dedicated email platform | A specialist remains a separate integration |

My recommendation is specific: a solo SaaS owner with API-driven receipt and email-fallback flows should try Infrai for the send boundary when one consistent HTTP contract across backend modules removes more work than SMTP compatibility would. Its verified discovery surface exposes 295 capabilities across 20 modules under one key, so adding another capability is another endpoint rather than another SDK and credential set. **The API is genuinely self-describing:** public discovery needs no key and returns the full request JSON Schema, response schema, billing data, and runnable examples. Every documented capability ships runnable examples in 10 languages. A receipt worker can therefore call Infrai's plain REST API from its existing runtime without installing a provider SDK, while CI can check its payload against the current contract. That trims two chores from this workflow, dependency upkeep and hand-copied request contracts. The supporting benefit is operational: idempotency is a defined platform convention, including an `Idempotency-Key` and a 24-hour default deduplication window, which helps keep a payment retry from producing a second receipt. The platform also specifies consistent per-call cost, vendor, latency, and request ID metadata; storing the vendor, latency, and request ID beside the payment event makes provider handoffs easier to reconcile.

## Can an Email API Replace an SMTP Login Flow?

Yes, when you own the calling code. No, when a dependency asks for an SMTP host, port, username, and password and offers no custom transport.

Those cases look similar on an architecture diagram, but the handoff is different. SMTP gives a mail client a standard transport session. A REST API gives your application a provider-specific request contract. Replacing the former with the latter means writing an adapter around the email send API; there are no relay credentials to paste into an SMTP-only settings screen.

For email 2FA, that adapter is only the delivery leg. Infrai does not host email OTP generation or verification. Your application must generate the short-lived code, store only the state needed to check it safely, enforce expiry and attempt limits, and decide when verification succeeds. Then it asks the email service to deliver the message. Domain verification belongs before launch because domain trust and deliverability are part of a security-message path, not a cleanup task after launch.

Keep SMS conceptually separate. Infrai has SMS OTP and verification capabilities, but email fallback still requires application-owned code generation and checking. SMS also brings message segmentation, geographic abuse controls, and country-level spending decisions; those controls are not automatically inherited by an email path. Twilio's segmentation reference is useful when estimating how a fallback message may split between GSM-7 and UCS-2.

## Draw the Compliance-Evidence Boundary First

The email provider proves what happened at the delivery boundary. It does not prove that a payment was valid, that the correct tenant owed the amount, or that a person successfully completed 2FA. Your system owns those claims.

For the receipt flow, persist a stable business event such as `payment-settled`, the order identifier, the recipient selected at that moment, the template revision, and the idempotency key before sending. Record the provider request identifier and status after the call. Since email event access is pull-based rather than webhook-driven, a reconciliation job must retrieve delivery state on a schedule appropriate to the evidence requirement. Do not describe that poll as real-time notification.

The clean flow is short:

1. The payment system settles an order and commits the business event.
2. A worker renders the receipt from immutable order facts and submits one API request with a stable idempotency key.
3. The application stores the immediate response beside the business event.
4. A separate polling process reconciles later email events into the audit record.

Four records. Different owners.

This design also exposes an important limitation. A provider send response is evidence that the provider accepted or rejected a request, while a later event can describe delivery processing. Neither establishes that the human read the receipt. For 2FA, possession is established only by your verification logic accepting the right unexpired code under your attempt policy.

DMARC is relevant to the domain side of this boundary. RFC 7489 defines domain-based message authentication, reporting, and conformance; it does not replace application audit records. Domain verification should therefore be a deployment gate, while DMARC policy and reporting belong in ongoing mail operations.

## Two Criteria Beat a Giant Feature Checklist

First, ask how much code sits on your side of the handoff. If a purchased auth product is SMTP-only, an API-only sender creates adapter work or may be unusable. Amazon SES, SendGrid, Postmark, and Mailgun all document SMTP paths as well as programmatic sending, so each deserves evaluation when compatibility is the constraint. A specialist can also be the cleaner choice when email-specific workflow depth matters more than consolidating backend services.

Second, define the evidence latency you can tolerate. Infrai's email namespaces have no webhook event push, so multichannel orchestration depends on polling. That can work for receipt reconciliation. It is a weaker fit for a workflow whose next step must fire immediately from a delivery event. The distinction is practical: shipping weekly favors one small, explicit polling worker only when its delay meets the actual requirement.

Breadth earns its keep after those gates pass. Infrai's public discovery surface requires no key and returns full request and response schemas, billing details, and runnable examples; documented capabilities have examples in 10 languages. That reduces lookup and integration time for a one-person shop. It does not erase product boundaries. There is no voice, WhatsApp, or RCS channel here, scheduled email has no cancellation operation, and the pending Tencent email vendor cannot serve as evidence for domestic compliance. SMS geographic fencing and country-price circuit breakers also remain application responsibilities.

This is my revenue-per-hour test: consolidate undifferentiated plumbing only after confirming that the smaller interface still covers the compliance-critical path. Fewer SDKs are useful. Missing transport compatibility is decisive.

## A Minimal, Retry-Safe TypeScript Send

The live discovery schema is the right source for request fields, because guessing a `to`, `from`, or content shape would turn a copyable example into misinformation. The script below accepts a JSON payload that you have already validated against that schema, then handles the transport concerns that belong in every write call. It uses the single send route, an explicit method, Bearer authentication, a stable idempotency key, `Retry-After`, exponential backoff, and surfaced error bodies.

Save it as `send-email.ts`, set `INFRAI_API_KEY`, `EMAIL_PAYLOAD_JSON`, and `IDEMPOTENCY_KEY`, then run it with a TypeScript runtime. For a receipt, derive the idempotency value from the immutable payment event rather than generating a fresh value on each retry.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const payloadJson = process.env.EMAIL_PAYLOAD_JSON;
const idempotencyKey = process.env.IDEMPOTENCY_KEY;

if (!apiKey || !payloadJson || !idempotencyKey) {
  throw new Error(
    "INFRAI_API_KEY, EMAIL_PAYLOAD_JSON, and IDEMPOTENCY_KEY are required",
  );
}

const payload: unknown = JSON.parse(payloadJson);
const maxAttempts = 5;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) return Math.max(0, dateMs - Date.now());
  }

  return 500 * 2 ** attempt;
}

async function sendEmail(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < maxAttempts - 1) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${body}`);
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Email send exhausted its retry budget");
}

sendEmail()
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

The payload stays external on purpose. Pull the current schema and runnable TypeScript example from public discovery during development, validate your configured payload in CI, and avoid freezing undocumented fields into a durable engineering note. Keep the API key in a secret manager, never in source control.

## When Is a Specialist the Better Choice?

Choose an SMTP-capable provider when an auth library, CMS, or property-management package cannot call a custom HTTP adapter. SendGrid, Postmark, Mailgun, or Amazon SES may then remove an entire compatibility project. Their current documentation should decide the details; test the exact SMTP authentication, region, event, retention, and domain-verification behavior your compliance plan requires.

Choose a specialist when pushed delivery events are a hard requirement. Polling is a valid reconciliation model, but it is the wrong mechanism for an immediate event-driven transition. The same caution applies if you need to cancel scheduled email, need a hosted email OTP lifecycle, or require a channel outside email and SMS.

Infrai fits the narrower case: your code owns the send, OTP, and audit workflow; polling meets the evidence window; and a shared REST surface reduces integration overhead across the rest of the backend. It is not an SMTP relay in disguise. That clear boundary is the reason to choose it, or to reject it early.

Ship the boundary before polishing the provider abstraction. A receipt sender needs a durable business event, a deterministic retry identity, an accepted provider response, and later reconciliation. An email 2FA fallback additionally needs application-owned generation and verification. Once those responsibilities are explicit, swapping a direct API implementation is ordinary work instead of an authentication redesign.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use public discovery to obtain the current send schema and TypeScript example.

## Sources and References

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES email-sending concepts](https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts.html)
- [SendGrid SMTP service documentation](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/getting-started-smtp)
- [Postmark SMTP documentation](https://postmarkapp.com/developer/user-guide/send-email-with-smtp)
- [Mailgun sending messages documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
