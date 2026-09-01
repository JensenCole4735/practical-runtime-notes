# Why Node.js Webhook Retry Queues Duplicate Delayed Jobs

A game renewal reminder has a hard business deadline: send it after the player's renewal window opens, but recover cleanly if either the worker or the receiving webhook disappears around that moment. That recovery requirement changes the design. Use a durable delayed-job queue with at-least-once delivery, then make the webhook effect idempotent at the business boundary.

Short answer: duplicate webhook processing happens because a worker can finish the remote side effect but lose its acknowledgement; the queue must then retry a job whose outcome it cannot prove. Idempotency is what makes that necessary retry harmless.

This is not an edge case to wish away. It is the normal ambiguity created by two systems that cannot commit one atomic transaction. The practical target is not exactly-once execution. It is one observable business effect despite one or more delivery attempts.

## Why does a Node.js webhook retry queue duplicate delayed jobs?

Picture renewal `renewal_7f31`, due at `2026-08-14T09:00:00Z`. A worker claims its delayed job, signs and sends the webhook, and the receiver records the reminder. Before the worker records success, its process exits or its acknowledgement is lost. From the queue's point of view, the job is unfinished. Returning it to the retry queue is the recoverable choice, so a second worker may send the same logical reminder.

Both systems behaved reasonably.

At-least-once delivery means an item may be delivered again until completion is acknowledged. A retry can also follow a network timeout where the sender genuinely does not know whether the receiver committed the request. Retrying only explicit failures misses that ambiguous outcome. Treat timeouts, lost acknowledgements, worker restarts, and expired leases as possible redeliveries rather than evidence that the first attempt had no effect.

The distinction between a job ID and an idempotency key matters here. A job ID identifies a queue record. An idempotency key identifies the business effect. If an operator recreates a failed queue record, or a producer publishes the same renewal twice, two job IDs can still refer to one reminder. A stable key derived from the event identity and action, such as `renewal_7f31:renewal-reminder`, survives those operational paths.

The receiver owns the final guarantee because it owns the side effect. Sender-side deduplication saves traffic, but it cannot close the gap between the receiver committing and the sender observing the response. This boundary is the constraint that determines the architecture.

## How should delayed webhook jobs enforce idempotency in Node.js?

The useful contract is small: persist a scheduled job, lease due work, send a signed request with a stable idempotency key, and acknowledge only after a successful response. The queue implementation can be hosted or self-managed; the correctness properties belong in the application contract.

That is the contract.

```ts
type RenewalReminder = {
  jobId: string;
  renewalId: string;
  playerId: string;
  dueAt: string;
  webhookUrl: string;
};

type Queue = {
  schedule(job: RenewalReminder): Promise<void>;
  leaseDue(now: Date): Promise<RenewalReminder | null>;
  acknowledge(jobId: string): Promise<void>;
  retry(jobId: string, nextAttemptAt: Date): Promise<void>;
};

type DeliveryResult = {
  ok: boolean;
  retryAfterMs?: number;
};

function reminderKey(job: RenewalReminder): string {
  return `${job.renewalId}:renewal-reminder`;
}

async function deliver(
  job: RenewalReminder,
  secret: string,
): Promise<DeliveryResult> {
  const body = JSON.stringify({
    type: "renewal.reminder_due",
    renewalId: job.renewalId,
    playerId: job.playerId,
    dueAt: job.dueAt,
  });
  const signature = await sign(body, secret);
  const response = await fetch(job.webhookUrl, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": reminderKey(job),
      "x-webhook-signature": signature,
    },
    body,
    signal: AbortSignal.timeout(10_000),
  });

  if (response.ok) return { ok: true };
  return { ok: false, retryAfterMs: 30_000 };
}

async function runOnce(queue: Queue, secret: string): Promise<void> {
  const job = await queue.leaseDue(new Date());
  if (!job) return;

  try {
    const result = await deliver(job, secret);
    if (result.ok) {
      await queue.acknowledge(job.jobId);
      return;
    }

    await queue.retry(
      job.jobId,
      new Date(Date.now() + (result.retryAfterMs ?? 30_000)),
    );
  } catch {
    await queue.retry(job.jobId, new Date(Date.now() + 30_000));
  }
}

async function sign(body: string, secret: string): Promise<string> {
  const key = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"],
  );
  const bytes = await crypto.subtle.sign(
    "HMAC",
    key,
    new TextEncoder().encode(body),
  );
  return Buffer.from(bytes).toString("hex");
}
```

The receiver should claim the idempotency key and apply the reminder state in one database transaction. A unique constraint, rather than a read followed by a write, closes the race between simultaneous deliveries.

```ts
type ReminderPayload = {
  renewalId: string;
  playerId: string;
  dueAt: string;
};

async function acceptReminder(
  db: Database,
  idempotencyKey: string,
  payload: ReminderPayload,
): Promise<"applied" | "duplicate"> {
  return db.transaction(async (tx) => {
    const claimed = await tx.tryInsertDelivery({
      idempotencyKey,
      eventType: "renewal.reminder_due",
    });
    if (!claimed) return "duplicate";

    await tx.markRenewalReminderDue({
      renewalId: payload.renewalId,
      playerId: payload.playerId,
      dueAt: payload.dueAt,
    });
    return "applied";
  });
}

type Database = {
  transaction<T>(work: (tx: Transaction) => Promise<T>): Promise<T>;
};

type Transaction = {
  tryInsertDelivery(input: {
    idempotencyKey: string;
    eventType: string;
  }): Promise<boolean>;
  markRenewalReminderDue(input: ReminderPayload): Promise<void>;
};
```

Return success for a previously applied key. A duplicate is not a receiver failure, and making the sender retry it forever turns a solved ambiguity into noise. Also retain the original result if the response body matters; the same key should not quietly perform a different operation with different input.

## Test recovery before choosing queue machinery

A delayed queue is only useful if an operator can answer four questions after a bad deploy or an interrupted worker: what is due, what is leased, what will retry, and what is dead-lettered. Record the stable business key, attempt count, next-attempt time, last outcome category, and correlation ID. Avoid logging secrets or full player payloads. Retries need limits. Exponential backoff with jitter reduces synchronized retry bursts, while a dead-letter state stops permanently rejected work from consuming the queue forever. The exact attempt count and delay are policy choices, not universal constants. Set them from the renewal deadline and the receiving system's recovery expectations. I'm not sure there is a defensible default without those two inputs; a reminder useful for hours permits a different schedule from one that expires in minutes. Ship the recovery path with the first version. Test the ambiguity directly: let the receiver commit, prevent the worker from recording its acknowledgement, then release the lease and deliver again. The assertion is one reminder effect and two accepted attempts. Also test two workers racing for the same logical key, a retry after process restart, a payload reused with the wrong key, and a job that crosses its useful deadline. Observability should report business outcomes, not only worker activity. Queue depth and attempt counts can look healthy while duplicate reminders reach players. Track unique reminder keys applied, duplicate keys accepted, age past `dueAt`, dead-letter volume, and the time from business deadline to the first successful effect. Those signals make a weekly shipping cadence possible without turning every queue change into an archaeological project.

Then test the destination boundary.

Outbound webhooks create an SSRF risk because the worker fetches a destination influenced by another party. OWASP recommends allowlisting trusted destinations where that is possible and separating validation from the request itself. For a gaming platform with registered partner endpoints, validate the scheme and hostname at registration, reject loopback and private network destinations, resolve DNS under a controlled policy, and restrict redirects. Don't let retry logic become a path into internal services — recovery code still handles untrusted input.

## What should change as delayed-job volume grows?

Start with one queue and a database unique constraint. Split queues only when workloads have meaningfully different deadlines or failure domains. Renewal reminders should not wait behind a flood of low-priority telemetry, but premature queue topology adds dashboards, replay procedures, and deployment surface that steal revenue-producing hours.

At higher volume, batch producers, partition work by a stable key when ordering matters, and keep consumers horizontally replaceable. Preserve the idempotency record long enough to cover the maximum replay window. Storage is part of the trade: retaining keys forever is simple to reason about but grows without bound; expiring them too early permits an old replay to repeat the effect. Define a replay horizon, document it, and make cleanup follow that policy.

An outbox is worth adding when scheduling must be atomic with a local business transaction. Write the renewal change and an outbox row together, then have a relay publish the delayed job. Without that link, an application can commit the renewal but fail before scheduling its reminder. The catch is another table, relay, lag metric, and replay procedure. For a one-person SaaS, I would pay that operational cost only when a missed reminder has enough business impact to justify it.

Keep direct database scheduling when one process owns the workload, throughput is modest, and the team can implement leasing and recovery correctly. Use a dedicated job system when durable delay, concurrency control, retry state, and inspection would otherwise become a homegrown subsystem. BullMQ is one documented Node.js example of a Redis-backed queue with delayed jobs and retries, but choosing it still means operating its dependencies and understanding its delivery behavior. The brand is secondary. The recovery model is the decision.

## Cost boundaries and the weekly shipping rule

| Choice | Good fit | Not suitable when |
| --- | --- | --- |
| Database table with leases | Small workload, existing transactional database, minimal infrastructure | Queue operations or isolation needs exceed what the team can safely maintain |
| Dedicated job queue | Many delayed jobs, worker concurrency, retry inspection, separate workload controls | Its extra service and operational surface cost more than the workload warrants |
| Receiver idempotency record | The receiver controls the business effect and has transactional storage | The effect cannot be made conditional on a durable claim |
| Sender deduplication only | Reducing accidental duplicate publication | Preventing duplicates after an ambiguous webhook response |

My shipping rule is blunt: outsource the undifferentiated queue mechanics once maintaining them costs more revenue-per-hour than the product work they protect. Keep the idempotency contract in application code either way. No queue vendor can infer that `renewal_7f31:renewal-reminder` represents one business effect, and no delivery claim removes the receiver's transaction boundary.

The result is modest but dependable: schedule against the business deadline, assume redelivery, make the effect idempotent, and rehearse recovery before launch. Exactly-once execution is the wrong promise. One observable renewal reminder is the useful outcome.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- https://docs.bullmq.io/
