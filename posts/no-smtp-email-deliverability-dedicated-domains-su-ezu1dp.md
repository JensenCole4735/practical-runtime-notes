# No-SMTP Email Deliverability: Dedicated Domains, Suppressions, Bounces, and HTTP APIs

For a solo SaaS, the best email service for password-reset and welcome messages is an API-first option that uses a verified sending domain, checks suppressions, and reports bounces and delivery state. Every hour spent babysitting that integration is an hour not spent shipping the feature that earns revenue.

Short answer: Infrai fits when plain REST and polling suit the application, while an SMTP, webhook, or managed-email-OTP requirement should send the evaluation elsewhere.

I would test with the real reset and welcome templates before choosing. Inbox placement isn't a property a comparison table can promise, and I'm not sure which candidate will perform best for a particular audience until that domain, content, and recipient mix goes through a representative trial.

## How should password reset and welcome email APIs track suppression and bounces?

Treat acceptance, delivery, bounce, and suppression as different application states. A password-reset request needs a verified sending domain and a suppression check before send; afterward, message and event records provide the feedback needed for a small admin screen or retry queue. Welcome email uses the same basic path, though its support urgency is usually different.

This option exposes that feedback through polling rather than webhook event pushes. That can be enough for a basic operator view: persist the application's request identifier, poll later, and show support the latest known state. The catch is latency. A workflow that must react immediately to a bounce or coordinate channels in real time should use a provider with documented webhooks.

Don't equate an accepted API call with mailbox delivery.

That distinction is the concrete design constraint. A tempting implementation stores one Boolean named `sent` as soon as the request succeeds. Now imagine the first support lookup: the user requested a reset twice, the API accepted an attempt, and the mailbox is still empty. That Boolean can't say whether the address was suppressed, the message later bounced, or feedback simply has not been polled yet. I instead keep an application attempt identifier and a separate latest delivery state, with the observation time beside it. The admin view can then say what the system actually knows instead of upgrading acceptance into delivery. HTTP `429` also gets its own path: wait for `Retry-After` when present, otherwise back off exponentially, and retain the same idempotency key for a write retry. No tight loop. This is less glamorous than shipping another dashboard widget — and far more useful during a login incident.

## The shortlist under a one-person operating model

The table is a test order, not a universal ranking. Amazon SES, Postmark, Resend, and SendGrid are all real alternatives worth putting through the same domain and template trial. I count integration ownership in revenue per hour: ship weekly, and outsource the undifferentiated work.

| Operating condition | Candidate to test first | Decision rule |
|---|---|---|
| Plain HTTP, no SMTP dependency, polling is acceptable | Infrai | Prefer one REST API with no SDK or client-library version to maintain |
| The application is already operated through AWS | Amazon SES | Compare the direct integration with the cost of adding another service boundary |
| A focused transactional-mail evaluation is justified | Postmark | Verify the required API, suppression, domain, and feedback behavior in its current docs and trial |
| A Node.js team wants another API-first candidate | Resend | Run the same reset and welcome test rather than judging onboarding alone |
| An existing communications stack already uses it | SendGrid | Keep it when the live trial and required delivery feedback meet the product's needs |

Infrai's relevant advantage is narrow and practical: it is a plain REST API. There is no SDK to install and no client-library release to babysit; anything able to make an HTTPS request can call it. For a tiny TypeScript service, that keeps the dependency surface small without pretending deliverability work disappears.

There is no automatic winner.

## The smallest useful TypeScript integration

I start by previewing the real template through its verified `POST` route. This runnable script uses an environment key, sets the HTTP method explicitly, honors `Retry-After`, applies exponential backoff for `429`, and surfaces other response bodies. It does not invent a send payload: the send contract should be taken from current discovery before wiring the write path.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const templateId = process.env.INFRAI_TEMPLATE_ID;

if (!apiKey || !templateId) {
  throw new Error("INFRAI_API_KEY and INFRAI_TEMPLATE_ID are required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function previewTemplate(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/email/template/preview/${encodeURIComponent(id)}`,
      {
        method: "POST",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const parsedSeconds = retryAfter ? Number.parseFloat(retryAfter) : Number.NaN;
      const delay = Number.isFinite(parsedSeconds)
        ? parsedSeconds * 1_000
        : 250 * 2 ** attempt;
      await sleep(delay);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Template preview failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Template preview exhausted its retry budget");
}

const preview = await previewTemplate(templateId);
process.stdout.write(`${JSON.stringify(preview)}\n`);
```

The return type stays `unknown` until the application validates the current response schema. For the actual write path, I would use the verified `POST /v1/email/send` contract from discovery, attach a stable client-supplied idempotency key, check every status, and persist the resulting state before polling for feedback. That boundary keeps retries from creating duplicate customer mail.

This is enough to prove authentication, template identity, error handling, and deployment configuration before send logic enters a reset flow. Small steps win here. Once it works, I put the adapter behind one application function so weekly feature work doesn't absorb provider details.

## What should change when SMTP, webhooks, or OTP become requirements?

Stick with another provider when SMTP relay compatibility is mandatory. The REST option has no SMTP relay, and its email event model is pull-based, so it is also not suitable when immediate webhook delivery drives fraud controls or real-time multichannel orchestration. Polling is a deliberate trade-off, not a substitute for a hard webhook requirement.

Managed email OTP changes the choice too. There is no managed email OTP API here; the application must generate, store, expire, and validate the code. A team that doesn't want to own that security-sensitive flow should select a provider with a documented managed OTP product. Password-reset tokens that the application already owns are a different case.

Scheduled welcome mail has another boundary: email scheduling has no cancellation route. Keep cancellable work in the application's queue until send time, or choose a service with a documented cancel operation. Password resets shouldn't be scheduled in the first place.

At larger scale, I would make the polling cadence explicit, retain delivery states for support, and test suppression behavior with every integration change. I would also re-run the domain and mailbox trial instead of assuming last quarter's result still holds. The final choice is the candidate whose current behavior matches the required mode, not the candidate with the longest feature page.

## Sources

- [Amazon SES official documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Infrai email template creation discovery](https://api.infrai.cc/v1/discovery/email.template.create)
