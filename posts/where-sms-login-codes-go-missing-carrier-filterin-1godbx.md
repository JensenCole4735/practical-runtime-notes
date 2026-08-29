# Where SMS login codes go missing: carrier filtering, sender registration, delivery budgets

Pick one registered sender per country, give every SMS OTP a delivery deadline, and put a second channel behind that deadline. That is the least complex setup that survives contact with real carriers. Carrier filtering, an unregistered sender identity, shared routes and anti-fraud controls all sit between your API call and a handset in the US or EU, and none of them show up in your application logs — the send request returns 202 either way. A 2FA login built on the assumption that "accepted" means "arrived" will fail quietly, and it will fail worst for the users you can least afford to lose.

The system I have in mind through all of this: a property-management marketplace. Landlords post a work order, local vendors claim it, and the vendor gets an SMS when a new order lands. The same phone number, the same route and the same registration then carry that vendor's login code. One channel, two message classes, one reliability problem.

| Design | What it buys | Where it breaks |
|---|---|---|
| Shared route, no registered sender | Working in an afternoon | US A2P filtering drops or throttles it; failures are silent |
| Registered sender per market | The best odds SMS can give you | Registration lead time and per-country paperwork before launch |
| SMS with a timed fallback to another channel | Route-level failure stops being a support ticket | More state: dedupe, expiry, two audit trails |
| App-generated codes or passkeys, SMS for recovery only | Carriers leave the critical path entirely | Onboarding friction for non-technical users |

For a marketplace where a missed order alert costs someone money, I'd register a sender in every launch country and run SMS as the first rung of a ladder, not as the whole ladder. For staff and admin accounts — the ones with access to payouts — I'd skip SMS as the primary factor and use app-generated codes. The trade-off is honest: you now operate two authentication paths instead of one, and the vendor onboarding flow grows a step.

## What actually stops an SMS login code: carrier filtering, sender registration, or shared routes?

Four different mechanisms, and they need different fixes.

Application-to-person traffic in the US runs through carrier registration. An unregistered or under-registered sender on a long code gets filtered — not bounced with a helpful error, filtered — and the aggregator often still reports the message as accepted. Twilio's A2P 10DLC documentation lays out the brand and campaign registration model in detail, including the throughput tiers that come with it. Registration is not a formality you can defer past launch; it's the difference between delivery and a black hole.

Europe is not one route. Sender ID rules, alphanumeric sender support and pre-registration requirements differ per country, so "we tested it in Germany" tells you very little about Spain. The practical move is to keep an explicit table: country, sender identity in use, registration status, owner. Not a wiki page. A table your code can read, so a launch in a new country trips a check instead of a support ticket three weeks later.

Then there's the shared route. Shared short codes and pooled sender identities mix your traffic with everyone else's, which means someone else's spam complaint changes your delivery odds. It's cheap and it's fine for low-stakes notifications. For a login factor it's a dependency you can't inspect, can't measure per-tenant, and can't fix from your side.

The fourth mechanism is anti-fraud, and it cuts both ways. Carriers and aggregators filter traffic patterns that look like SMS pumping — artificially inflated traffic where a bot requests codes to premium destinations. Your own login endpoint is the attack surface. If an unauthenticated caller can trigger a message to any number in any country, you're paying for someone else's revenue scheme and degrading your own sender reputation while you do it. Country allowlists, per-number and per-account caps, and a challenge lock after repeated attempts are not fraud-platform features. They're a hundred lines of server code and an index.

Handset conditions round it out. Phone off, number ported, prepaid balance zero, message blocked by an on-device filter. Nothing on your side can fix those. That is exactly why the deadline and the fallback exist.

## Two criteria that decide the failure rate: sender identity and delivery accounting

Sender identity first, delivery accounting second. Everything else is downstream.

Sender identity is a registration and provisioning problem with a lead time measured in days or weeks, so it belongs in the launch plan for each market rather than in a sprint. Get it wrong and no amount of retry logic helps — you are retrying into the same filter. Get it right and most of the remaining loss is genuine handset failure, which is at least measurable.

Delivery accounting is the part solo teams skip, and it's the part that pays. An accepted send is a state, not an outcome. The states worth persisting are accepted, delivered, undelivered and unknown, per message, with the destination country and the time each transition arrived. From those four states you get the two numbers that actually drive decisions: accepted-to-delivered ratio per country, and time-to-delivery at the median and the tail. When a vendor in one market stops claiming work orders, you want to know within a day whether their delivery ratio moved, instead of guessing.

Providers expose that state either by pushing delivery receipts to a webhook or by letting you poll message status. Push gives you lower latency; poll gives you fewer moving parts and no public endpoint to secure. For a one-person team, poll first — a worker that reconciles open challenges every few seconds is easier to reason about at 2am than a webhook you can't replay. Move to receipts when the polling volume actually costs you something.

Two operational details make this real rather than decorative. Keep the provider's message ID next to your own challenge ID, or you cannot answer a support question at all. And check the state of the first message before you send a second one — a user staring at a delayed code and a script farming codes produce the same click pattern, so the decision has to come from stored state, not from a button timer in the browser.

## The ledger and the ladder, in one worker

Here's the shape I'd build it in. A gateway interface with no vendor in it, a challenge record that owns the deadline, and one decision function that the login endpoint and the reconciliation worker both call.

```ts
type Delivery = "accepted" | "delivered" | "undelivered" | "unknown";

interface Gateway {
  send(to: string, text: string): Promise<{ messageId: string }>;
  status(messageId: string): Promise<Delivery>;
}

interface Challenge {
  id: string;
  country: string;          // ISO 3166-1 alpha-2, from the destination number
  messageId?: string;
  state: Delivery;
  sentAt: number;
  attempts: number;
}

const DELIVERY_BUDGET_MS = 30_000;   // how long SMS gets before the ladder moves on
const MAX_ATTEMPTS = 3;

async function pollStatus(base: string, messageId: string): Promise<Delivery> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${base}/messages/${encodeURIComponent(messageId)}`, {
      method: "GET",
      headers: { authorization: `Bearer ${process.env.SMS_API_TOKEN ?? ""}` },
    });

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 400 * 2 ** attempt;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }
    if (!res.ok) return "unknown";

    const body = (await res.json()) as { state?: Delivery };
    return body.state ?? "unknown";
  }
  return "unknown";
}

type Next = "wait" | "retry_sms" | "escalate" | "lock";

function decide(c: Challenge, now: number): Next {
  if (c.state === "delivered") return "wait";
  if (c.attempts >= MAX_ATTEMPTS) return "lock";
  if (now - c.sentAt < DELIVERY_BUDGET_MS) return "wait";
  return c.state === "undelivered" ? "retry_sms" : "escalate";
}
```

`decide` is deliberately boring and pure, which means it's the one piece you can unit-test without a provider account: feed it a state and a clock, assert the rung. Everything above it — the worker cadence, the country allowlist, the per-number cap — is plumbing around that decision. The escalate branch is where a second channel takes over: email, an in-app prompt, or a support path that verifies the vendor another way.

Test it against a real handset in every launch country before you trust it. A staging number on a test route proves the code compiles, not that a carrier will deliver.

## When SMS is the wrong primary channel

SMS is not suitable as the primary factor when the account holds money or when your users are technical enough to install anything. App-generated codes over TOTP remove carriers, registration and roaming from the picture completely, and passkeys remove the shared secret as well. National guidance on digital identity has treated phone-network delivery as an out-of-band method with documented risks — number takeover, device loss — for years now, and that assessment hasn't gotten friendlier.

Stick with SMS as the primary when your users are exactly the marketplace vendors in my example: a plumber with an aging Android phone who will not install an authenticator app to accept a work order, and who is already receiving your order alerts on the same number. Reachability beats theoretical strength when the alternative is that they don't log in at all.

Email deserves a flag of its own. It looks like a free fallback and it isn't: you own code generation, storage, expiry and verification, plus a whole second deliverability problem with its own filtering rules and sender authentication — the SES documentation is a reasonable starting point for how much of that is your job rather than the provider's. Recovery codes issued at signup are often the cheaper answer for a small team, because there's nothing to deliver at all.

My release gate for a new market is short. Sender registered and recorded in the country table. Delivery states persisted with the provider's message ID. A budget, a ladder and a lock on the server rather than in the browser. One real handset test in-country. Ship it after that, and let the delivery ratio per country tell you what to fix next.

## Further reading

- [Twilio: US A2P 10DLC compliance documentation](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 6238: TOTP, Time-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc6238)
- [MDN: Web Authentication API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)
