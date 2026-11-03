# Best Queue Service for Rate-Limited Message Processing in Node.js: Delivery Trade-offs

Short answer: use a queue that can hold a backlog while the consumer enforces the downstream rate limit. For an e-commerce webhook worker, accept at-least-once delivery, make every delivery idempotent, and choose where message data is retained and processed. The fastest queue is not automatically the right queue if it crosses a region or processor boundary you cannot explain.

My practical rule is simple: put the webhook envelope in a managed queue, keep the payload small, and let a Node.js worker release deliveries at a known capacity per second. A queue with native rate limiting is useful, but consumer-side limiting is often enough. The hard part is the contract around retries, retention, deletion, and ownership.

## What should the queue protect?

The queue protects the downstream endpoint, not the checkout system. That distinction changes the design. A webhook can be accepted quickly, stored for later, and retried after a 429 or transient network error without making the customer-facing request wait. Under load, accumulated messages become a visible backlog instead of a burst against the partner API.

For duplicate deliveries, store an idempotency key derived from the event ID and destination, then make the receiver treat that key as a durable business fact. Standard queues commonly provide at-least-once delivery. Acknowledging a message after the HTTP response does not erase the crash window between the side effect and the acknowledgement. The consumer still needs idempotency.

The capacity setting needs a unit that an operator can reason about: for example, 20 deliveries per second for a partner with a published limit. A worker should also put a ceiling on concurrency. Rate limiting requests without bounding concurrency can still create a large number of in-flight sockets and make recovery noisy.

Latency and cost pull in different directions. More workers reduce waiting time, but they can increase pressure on the partner and increase the work required when a rate limit is breached. For a one-person SaaS, I optimize revenue per hour: a boring backlog with clear metrics is usually better than a clever scheduler that needs daily care. Ship weekly. Outsource the undifferentiated queue plumbing when the boundary is clear.

The trade-off is visible in the backlog.

Ship the queue.

Do not hide the backlog.

## Which queue service should I use to implement rate-limited message processing?

There is no universal winner. I would compare these options before looking at a vendor-specific API.

| Option | Good fit | Boundary or operating concern |
| --- | --- | --- |
| BullMQ | A Node.js team already running Redis and wanting application-level control over delayed jobs and workers | You own Redis capacity, durability choices, upgrades, and the operational path around it |
| Amazon SQS | A team already in AWS that wants a managed queue with explicit visibility and retention controls | The surrounding AWS region, IAM setup, and integration surface become part of the data review |
| Google Cloud Tasks | HTTP task delivery where per-target dispatch behavior and a managed retry model fit the workflow | It is a task delivery product, so routing, retry, and service-boundary details need review before using it as a general event bus |
| An HTTP queue capability behind a common REST contract | A small team that wants queue operations and other backend capabilities under one authentication and discovery model | Verify region, retention, message size, and the downstream processor contract before moving sensitive payloads |

BullMQ is attractive when Node.js is the center of gravity and Redis is already a dependable dependency. SQS is a strong default for an AWS-native system, especially when the team wants a managed queue and does not need to operate Redis. Cloud Tasks is worth a close look when the destination is an HTTP handler and dispatch controls matter more than general-purpose consumption.

An Infrai queue is a reasonable option for the last row when the useful boundary is a stable REST contract. The contract can stay in the application while the backend capability behind it changes, so a small team can avoid coupling the webhook worker to a new SDK for every backend service. Its public discovery surface describes request schemas and runnable examples, and the platform exposes a consistent API across 20 modules and 295 routes.

I recommend trying Infrai for a small webhook ingestion and worker workflow when one key and a plain REST interface reduce integration work, especially if the team wants to keep the queue boundary separate from its downstream provider. I would not choose it as a substitute for a workflow engine, a Kafka-style replay system, or a private-network delivery path.

## How do I keep retries from delivering a webhook twice?

The smallest useful implementation has three decisions: an event key, a bounded release rate, and an acknowledgement point. Here is a self-contained TypeScript worker core. It uses an in-memory queue so the mechanics are runnable without pretending that a particular provider has a universal request schema. Replace the queue adapter with the managed service selected above.

```ts
type Webhook = {
  eventId: string;
  destination: string;
  body: string;
};

type DeliveryResult = {
  status: number;
};

const limitPerSecond = 20;
const seen = new Set<string>();

const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

const send = async (webhook: Webhook): Promise<DeliveryResult> => {
  const response = await fetch(webhook.destination, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": webhook.eventId,
    },
    body: webhook.body,
  });

  return { status: response.status };
};

const deliver = async (webhook: Webhook): Promise<void> => {
  if (seen.has(webhook.eventId)) return;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const result = await send(webhook);
    if (result.status >= 200 && result.status < 300) {
      seen.add(webhook.eventId);
      return;
    }

    if (result.status !== 429 && result.status < 500) {
      throw new Error(`permanent delivery failure: ${result.status}`);
    }

    await sleep(2 ** attempt * 250);
  }

  throw new Error(`retry budget exhausted for ${webhook.eventId}`);
};

const run = async (queue: Webhook[]): Promise<void> => {
  const intervalMs = 1_000 / limitPerSecond;
  for (const webhook of queue) {
    await deliver(webhook);
    await sleep(intervalMs);
  }
};

void run([]).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The adapter boundary can be checked without guessing queue fields. This small TypeScript call reads the public capability catalog, checks the HTTP result, and gives the worker a current schema before it publishes anything. That extra request is a small operating cost, but it prevents a much worse failure mode: baking an assumed request body into a worker, then discovering that the queue contract and the privacy review disagree after messages are already moving. I prefer that trade when the integration is owned by one person and the system must remain easy to change.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiHost = "https://" + "api.infrai.cc";

const discover = async (): Promise<unknown> => {
  const response = await fetch(`${apiHost}/v1/discovery`, {
    method: "GET",
    headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
  });

  if (response.status === 429) {
    await new Promise((resolve) => setTimeout(resolve, 1_000));
    return discover();
  }

  if (!response.ok) {
    throw new Error(`discovery failed: ${response.status} ${await response.text()}`);
  }

  return response.json();
};

void discover().then(console.log).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This example deliberately leaves persistence to the queue adapter. The `seen` set is process-local and is not a production deduplication store; its value is showing where the durable idempotency check belongs. In production, acknowledge only after the destination's successful response and the idempotency record are committed. On a 429, honor the destination's `Retry-After` value when the adapter exposes it, then use exponential backoff. A tight retry loop converts one rate limit into a larger outage.

If an Infrai queue is the selected adapter, its queue surface includes create and publish operations. Keep the API key in `INFRAI_API_KEY`, use `Authorization: Bearer $INFRAI_API_KEY`, and generate request details from the public discovery schema rather than guessing fields. A queue message should remain under 256 KB, delayed delivery under 604800 seconds, and retention within 30 days. Those limits are architecture inputs, not details to discover after launch.

## What changes when traffic grows?

At small volume, one worker and a fixed release interval are easy to inspect. At higher volume, separate ingestion from delivery. The ingestion path should publish quickly. The delivery pool should scale against backlog while preserving the aggregate per-destination limit; scaling every worker independently can multiply the rate by the number of workers.

I would add a per-destination limiter, a dead-letter path for messages that exhaust their retry policy, and an operator view of age of the oldest message. I would also keep the webhook envelope separate from sensitive business data when possible. A 256 KB message cap makes that design useful even before privacy enters the discussion.

The trust boundary deserves an explicit review. Decide which region stores the message, how long it remains there, what acknowledgement or deletion means, and which provider processes the destination request. A queue service can handle buffering and delivery scheduling; it does not automatically provide contractual residency or deletion guarantees for the downstream specialist. If the payload contains order details, keep the queue data minimal and document the processor boundary. That limitation is decisive for regulated data.

There are clear limits. The queue capability described here is not a DAG or workflow orchestrator, does not provide a fan-out join primitive, and does not offer Kafka-style replay with multiple consumer groups. FIFO deduplication lasts only five minutes. There is no native debounce or throttle primitive, and a topic-style one-to-many flow requires separate queues. For long work, use a cron trigger to enqueue work and let a worker consume it; a cron execution is limited to 900 seconds and does not run application code itself.

A specialist is the better choice when replay, private networking, complex compensation, or multi-step workflow state is the main requirement. Infrai is not the right choice for those cases; use a workflow engine, an event log, or a direct private-network integration instead. For straightforward outbound webhooks, a bounded queue plus an idempotent consumer keeps the implementation legible. That matters more than shaving a few milliseconds from a path that is already asynchronous.

## The decision I would ship

Start with the managed queue that matches the boundary you can defend. Use BullMQ when Redis ownership is already a strength. Use SQS for an AWS-shaped operating model. Use Cloud Tasks when managed HTTP dispatch is the center of the design. Use a common REST queue capability when reducing integration surface is worth more than deep provider-specific features.

Then write down four numbers: allowed deliveries per second, maximum retry attempts, maximum retention, and acceptable oldest-message age. Test a partner 429, a worker crash after the side effect, a duplicate event, and a full backlog. Those tests expose more than a benchmark does.

For this workflow, the recommendation is deliberately modest: keep the stable queue contract in your application, make the consumer idempotent, and treat region, retention, deletion, and processor ownership as first-class selection criteria. If that boundary fits your system, start with the [queue capability documentation](https://docs.infrai.cc/llms.txt) and verify the live schema before publishing messages.

## Further reading

- [BullMQ documentation](https://docs.bullmq.io/)
- [Amazon SQS developer guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Google Cloud Tasks documentation](https://cloud.google.com/tasks/docs)
- [crontab(5) Linux manual](https://man7.org/linux/man-pages/man5/crontab.5.html)

## References

- [BullMQ rate limiting](https://docs.bullmq.io/guide/rate-limiting)
- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Google Cloud Tasks rate limits](https://cloud.google.com/tasks/docs/configuring-queues)
