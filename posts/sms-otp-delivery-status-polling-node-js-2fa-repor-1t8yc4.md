# SMS OTP Delivery Status Polling: Node.js 2FA Report UX Without Webhooks in US/EU

Short answer: for a media SaaS that sends a generated report as an email attachment, poll SMS OTP delivery status only as an exception path, keep OTP templates and login policy in the application, and use a resend timer for the normal 2FA UX. A pull-based REST API is fine for an MVP. It is the wrong foundation for a flow that promises instant cross-channel failover.

The important boundary is template ownership. The browser owns the resend countdown. The server owns code expiry, attempt limits, and the decision to send an email fallback. The messaging provider reports what can be read, but without webhook events it cannot drive that decision in real time.

That arrangement supports weekly shipping. It also prevents a delayed delivery check from blocking report generation after the user has already authenticated.

## Build log: keep the login clock separate from the delivery clock

An OTP login has two clocks. The user sees a countdown and enters a code. The application may later inspect a provider status or event timeline. Those clocks should not be presented as the same thing.

Both email and SMS event models here are pull-based. Delivery confirmation and fallback decisions are therefore slightly delayed. For a simple SaaS, that is usually an acceptable trade. For an authentication product that promises immediate reaction to a carrier event, it is a product limitation.

I would start with four application states: `created`, `waiting`, `verified`, and `expired`. A diagnostic outcome such as `delivery_unknown` belongs in server logs and support tooling. It does not need to become the main browser state.

Do not poll every second. The countdown is local UX state, not evidence that a carrier delivered anything. Poll status when a user reports a missing code, when an exception workflow needs evidence, or when a bounded fallback decision is due. Then stop after a small retry budget. The concrete ceiling here is 4 attempts, with each backoff wait capped at 30 seconds.

Keep it boring.

Consider the case that causes the most confusion. A reader requests a media report, the service sends an OTP, and the first status check still says the delivery is pending. The user enters the valid code before the next check completes. Authentication has succeeded in the application, so the report job can now be released; waiting for a later carrier event would only make the user wait longer. If the code expires instead, the report job stays gated. If a fallback email is allowed, the application records that decision and owns the second template. The provider status is useful evidence for the exception path, but it is not the authority for session state. Keeping those decisions separate is the difference between a calm login screen and a progress indicator that pretends to know more than it does.

For US and EU traffic, country-specific anti-abuse limits and spending cutoffs belong in the application layer. They are business rules. The messaging layer cannot supply the geographic fence or the decision to stop repeated sends for a country.

## Which template boundary fits a report-email login?

The useful comparison is not a feature-count contest. It is a question of who owns the text, lifecycle, and failure decision.

| Option | Where it fits | Boundary to check |
| --- | --- | --- |
| Twilio Verify | A managed verification flow for a team willing to delegate more OTP mechanics | Confirm current channel, regional, template, and callback behavior before making failover a product promise |
| Amazon SNS | An AWS-oriented service that wants messaging primitives inside its existing account and observability model | Delivery feedback and OTP policy still need an application design for the selected channel and region |
| SendGrid | Report email or email-based fallback when email delivery is the primary artifact | It does not remove the need to own the login code lifecycle or decide how an SMS failure changes UX |
| Pull-based REST capability | A small service that wants language-neutral HTTP and application-owned templates | There is no webhook push, hosted email OTP interface, SMTP relay, or voice, WhatsApp, or RCS channel |

The last row is where Infrai can fit, but only under that boundary. Infrai is a plain REST API, so one key / one bill can cover the report and other backend capabilities instead of creating a new key and invoice for every small integration. A Node.js service has no SDK to install or client-library version to babysit. Its public, self-describing discovery surface also exposes capability schemas and runnable examples without a key, which makes it easier to verify a request shape before wiring credentials into a report job.

Its wider surface also matters to a one-person SaaS. The live discovery facts describe 295 routes across 20 modules under one key. A report workflow can keep one integration convention as it grows into other backend capabilities, rather than collecting a separate credential and client style for every small job. This reduces setup and reconciliation work; it does not turn pull-based delivery into push-based orchestration.

The template decision remains yours. A fallback email needs an application-owned OTP template because there is no hosted email OTP interface. The service also has no SMTP relay. Pick this model when the product values language-neutral integration and local policy control more than immediate delivery events.

## Should Node.js poll SMS status during a 2FA login?

Use a bounded server-side check. The message identifier comes from the send operation that the application owns. On the normal path, let the browser wait for the code and enforce the resend timer locally. On an exception path, read the current status and surface a real error instead of treating an unsuccessful response as delivery.

```ts
const apiBase = process.env.INFRAI_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase || !apiKey) {
  throw new Error(
    "Set INFRAI_API_BASE_URL to the versioned API base and INFRAI_API_KEY",
  );
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  if (Number.isFinite(seconds) && seconds >= 0) {
    return Math.min(seconds * 1000, 30_000);
  }
  return Math.min(500 * 2 ** attempt, 30_000);
}

export async function readSmsStatus(
  messageId: string,
  apiKey: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${apiBase}/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.ok) {
      return response.json();
    }

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    throw new Error(`SMS status request failed (${response.status}): ${body}`);
  }

  throw new Error("SMS status request exhausted its retry budget");
}

const messageId = process.argv[2];
if (!messageId) {
  throw new Error("Usage: INFRAI_API_KEY=... node status.js <message-id>");
}

readSmsStatus(messageId, apiKey).then((status) => {
  console.log(JSON.stringify(status));
});
```

The sample uses the verified status route, an explicit method, a bearer key from the environment, and a capped retry loop. It honors `Retry-After` for rate limits and reports a 4xx or 5xx response to the caller. The 4-attempt ceiling and 30-second wait cap are application policy, not a claim about carrier timing. The loop reads state; it does not resend a code, so a retry cannot create a second message.

The event timeline route is better for an operator investigating a sequence than for a browser that wants a live progress bar. Neither route creates a webhook. If the product needs to react immediately to a delivery event, this polling design cannot provide that signal.

The sample is deliberately not a complete OTP implementation. Code generation, hashing, expiry, replay prevention, session binding, and rate limits still belong in the application. OWASP's guidance is the right place to validate those security decisions.

## What changes when the report is the deliverable?

Separate authentication from report delivery. After the code is accepted, enqueue report generation, persist the attachment privately, and send the email as its own job. A pending carrier status should not hold a valid session open.

The awkward case is easy to miss: a user enters a valid code while a status poll still says `pending`. The application already has the authoritative authentication result. Waiting for a later carrier event adds delay without improving that decision. If the code expires, do not release the report job. If it succeeds, give the report job its own idempotent identity and retry path.

This is also where template ownership pays for itself. The application can keep the OTP message, resend message, report subject, and fallback email consistent without hiding a provider-specific template decision in a managed verification product. The cost is maintenance. Someone must update and test those templates, and someone must decide what a US or EU fallback is allowed to do.

Email scheduling needs the same separation. There is no email cancellation route in this capability set, so do not schedule a report email merely because an OTP check is pending. Gate the report job in application state, then send it once the session is valid.

## What I would change at scale

I would record the OTP attempt, message identifier, country-policy decision, resend count, and terminal application outcome. I would inspect the event timeline for diagnostics or an exception workflow, not turn every browser tab into a delivery monitor.

At higher volume, the absent push event becomes an attention problem. A cross-channel flow must ask what happened before it decides to send email. That delayed branch needs expiry, idempotency, and a maximum wait; otherwise an old SMS can arrive after the fallback email and confuse the user.

The decision rule is narrow: use bounded polling when a resend window and a human-readable exception path are acceptable. Choose a push-capable, more managed verification design when an immediate delivery event is part of the product promise. Keep template ownership in the application when that control is worth the ongoing work.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SNS SMS delivery status](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)
- [SendGrid Event Webhook documentation](https://docs.sendgrid.com/for-developers/tracking-events/event)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
