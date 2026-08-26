# One Login Code, Two SMTP Paths: OTP Auth Email Reliability

The reason not to use an SMTP relay for OTP email by default is that a short-lived login code and an asynchronous delivery handoff have different state boundaries. That constraint changes the architecture.

Short answer: don't add an SMTP relay or a mixed provider setup by default; keep one authoritative OTP state, one delivery adapter, and a measured failover policy, then add a second path only when its operational value exceeds the ambiguity it creates.

This is less about picking a transport than protecting the login state machine. A second sender can improve independence, but it can also produce duplicate messages, reorder codes, split telemetry, and turn an uncertain first attempt into two accepted attempts. For a one-person SaaS shipping weekly, that extra state has to earn its keep. The revenue-per-hour lens favors outsourcing commodity delivery while keeping authentication decisions inside the application.

## The constraint that changed the choice

SMTP is a handoff protocol. An accepted handoff is useful evidence, but it is not the same event as a user seeing a message. That gap matters more for OTP than for a receipt because the code expires, retries are common, and a late message may arrive after a newer code has been issued.

The tempting design is simple on a whiteboard: send through provider A, wait briefly, then use provider B. The hard question is what the timeout means. It may mean the first submission was rejected, delayed, accepted without a prompt response, or accepted while the client lost the response. Consider the ambiguous sequence rather than the happy path: a worker submits code A, its connection closes before it records a definitive result, and a retry worker submits through the other route. The first message can still be in motion. If the retry creates code B, the recipient may see A after B and naturally enter the latest-arriving message, even though the application considers B current. If the retry keeps code A, duplicate delivery is less confusing but still affects support, rate limits, and telemetry. Neither branch is automatically wrong. The mistake is allowing a network timeout to pick one without an explicit authentication policy.

Don't let transport choose authentication state.

Unknown is a state.

The application should create one challenge, store only the verification material it needs, and pass the same challenge identifier through every delivery attempt. A resend policy then decides whether the same still-valid code can be delivered again or whether a new challenge invalidates the old one. That is a product and security decision, not an SMTP callback.

Sender reputation and authentication also live above the individual request. Yahoo's sender guidance calls for authenticating mail and describes SPF, DKIM, and DMARC expectations, along with complaint-rate and subscription practices for relevant traffic. Mixing providers therefore means more than adding another credential: every authorized sender must fit the domain's authentication policy, and operational ownership must include suppression and complaint handling where applicable. An emergency path that was never exercised with the production domain is just an assumption.

## Should OTP email use an SMTP relay in a mixed provider setup?

Usually, no. Start with one provider path when login email volume and business risk do not justify operating independent delivery routes. A single path gives one place to inspect acceptance, latency, bounces, and configuration. It also keeps incident reasoning small enough to do while shipping the actual product.

A relay is reasonable when an existing mail layer already owns domain authentication, connection policy, and delivery telemetry. A mixed provider setup is reasonable when independence is a stated reliability requirement and the team can test it continuously. Neither becomes safer merely because two vendors appear in a diagram.

The catch is real: one path concentrates dependency risk. It is not suitable when email login is the only way into a high-value product and the business has a defined recovery target that one delivery path cannot meet. In that case, use a second preconfigured route or offer another authentication channel, but make the trigger explicit. Conversely, stick with one path when nobody has time to reconcile events, exercise failover, and maintain sender authentication across both routes.

I'm not sure a universal timeout can settle that trade-off. Your mileage may vary because the right threshold depends on the code lifetime, observed acceptance latency, user retry behavior, and the cost of a duplicate versus a delayed login. Those measurements, not a copied five-second constant, should resolve it.

## The smallest delivery contract that stays honest

I want the application to know only three transport outcomes: accepted, rejected, or unknown. “Accepted” means the selected delivery service accepted responsibility for the request. “Rejected” means it definitively did not. “Unknown” means the caller cannot prove either state. The last case is uncomfortable — and essential — because collapsing it into rejection makes unsafe retries look tidy.

Here is the shape I would ship first. It uses a generic adapter, keeps the challenge stable across attempts, and records enough correlation data to reconcile later events without letting a vendor response mutate the OTP.

```ts
type DeliveryOutcome =
  | { state: "accepted"; attemptId: string; providerMessageId: string }
  | { state: "rejected"; attemptId: string; reason: string }
  | { state: "unknown"; attemptId: string; reason: "timeout" | "connection_lost" };

type OtpMessage = {
  challengeId: string;
  recipient: string;
  code: string;
  expiresAt: Date;
};

interface EmailTransport {
  send(message: OtpMessage, attemptId: string): Promise<DeliveryOutcome>;
}

async function deliverOtp(
  transport: EmailTransport,
  message: OtpMessage,
  attemptId: string,
): Promise<DeliveryOutcome> {
  if (message.expiresAt.getTime() <= Date.now()) {
    return { state: "rejected", attemptId, reason: "challenge_expired" };
  }

  return transport.send(message, attemptId);
}
```

The `attemptId` must be unique for a submission, while `challengeId` remains stable for the authentication challenge. Store both before sending. That ordering lets a later delivery event attach to a known attempt, even if the request returned `unknown`. It also supports a simple rule: never launch a second route merely because the first call is slow; consult a policy that accounts for challenge age and prior attempt state.

Keep the email body boring. Include the code, its expiration meaning, and enough context for the recipient to recognize the login. Do not put the verification decision in a provider template callback. The auth service remains authoritative, rate-limits requests, compares the submitted code, expires the challenge, and prevents successful reuse.

Testing needs the same separation. Unit-test state transitions with a fake transport that returns all three outcomes. In integration tests, verify that the adapter preserves the challenge and attempt identifiers. In a deployment check, send to controlled inboxes and confirm the domain's authentication results and event ingestion. A successful API or SMTP call alone doesn't cover the user journey.

## What I would change at scale

At larger volume, I would place a durable queue between challenge creation and delivery, then make workers claim attempts idempotently. Provider-specific events would enter a normalized event log rather than updating the challenge directly. Dashboards would separate submission acceptance, bounce outcomes, and end-to-end login completion; combining them into one “delivery rate” hides where the system is failing.

Only then would I automate route changes. The policy needs a circuit state, a minimum evidence window, and a way to stop duplicate dispatches. It should also preserve domain alignment on every route. This costs engineering time, on-call attention, and periodic drills. Two paths without those controls create more branches, not dependable redundancy.

There is a smaller alternative: keep a single email route and provide recovery codes or another deliberately designed login method. That choice avoids cross-provider mail state, though it transfers complexity into enrollment, account recovery, and support. There is no free fallback.

Ship the smallest system whose failure you can observe and explain. Add redundancy after the state machine, domain authentication, event reconciliation, and drills are ready to carry it.

## Sources

- https://resend.com/docs/introduction
- https://senders.yahooinc.com/best-practices/
