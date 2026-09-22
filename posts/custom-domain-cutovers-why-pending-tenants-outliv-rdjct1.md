# Custom Domain Cutovers: Why Pending Tenants Outlive a Stopped Verification Job

| Choice | When it fits | Main cost |
| --- | --- | --- |
| Poll DNS in a durable, independently monitored queue | A school needs a domain cutover soon after its DNS change becomes visible | More state and worker operations |
| Run a periodic batch scan of pending domains | Cutover can wait until the next scan | A missed run can leave every tenant pending |

Short answer: for an edtech SaaS onboarding schools on their own custom domains, choose durable per-tenant verification attempts with a separate scheduler heartbeat and a maximum age for `pending`. Keep the batch scan as a recovery sweep. A successful DNS lookup proves only that the lookup happened; it does not prove the scheduled verifier will run again. When onboarding stalls, compare the age of the oldest pending request with the last completed verification run before asking customers to change records.

That comparison comes first.

## Why are custom domain tenants stuck pending after the scheduled job stops?

A tenant can remain pending for two very different reasons. Its required DNS record may not yet be visible to the resolver used for verification, or nobody has checked since it became visible. Treat those as separate clocks: time since the customer submitted the hostname and time since the verifier last attempted that hostname. A missing attempt is a scheduling problem, not evidence of slow propagation. To debug a tenant stuck forever, start with the last attempt timestamp rather than repeatedly refreshing a DNS lookup from a different network.

Check the exact owner name and record type expected by your domain-ownership flow, then query authoritative DNS as well as the resolver path your worker uses. DNS caches obey TTLs, but negative answers can also be cached according to the zone's SOA settings under RFC 2308. That means a recent record change and a stale negative cache result can coexist. Record the resolver, answer, and observation time; do not tell a school that propagation is complete based on a single query from your laptop. The DNS ownership challenge for a web hostname is also distinct from DMARC, which applies to email authentication policy. A valid `_dmarc` TXT record does not verify control of a classroom portal hostname.

This distinction decides the next action. If the expected record is absent at authoritative servers, show the tenant the exact name and value to configure. If it is present but the latest attempt predates the change, investigate dispatch. If the worker attempted verification recently and still reports a mismatch, inspect the answer and the expected target before changing any status. Three cases, three owners. Consider a school that edits its TXT record while a verifier is offline: the dashboard could still say pending after the record is visible, and repeatedly lowering a TTL won't restart the job. Compare timestamps before treating any DNS change as the fix; if the last completed run precedes the edit, the immediate operational issue is the missing run. If runs continued, check whether the worker queried the exact owner name and retained the answer, then distinguish a mismatched value from a cached negative response. This sequence prevents support from sending a school back to its registrar when the request has never reached a worker.

No lookup, no verdict.

## Cutover speed depends on the second clock

For a one-person SaaS shipping weekly, the expensive failure is a human support loop that cannot tell a DNS delay from a dead job. A periodic scan has a simple operational footprint, but its worst-case wait includes the scan interval and any missed runs. Per-tenant retry attempts can reduce the gap between visibility and activation, provided the retry state survives process restarts. Neither approach can force external DNS caches to refresh. The trade-off is operational: individual queued retries aren't suitable if nobody can monitor claims, leases, and stuck attempts; in that setting a monitored batch scan with an honest cutover expectation may be the better choice.

Keep the tenant state explicit: `pending`, `verified`, or `needs_action`, plus `createdAt`, `lastAttemptAt`, `nextAttemptAt`, and the last observed DNS outcome. A separate heartbeat records the time a scheduler cycle completed, including cycles that found no work. Alert when that heartbeat is overdue and when the oldest eligible pending tenant has passed the intended retry window. Those signals catch a stopped schedule even if no exception was emitted. A heartbeat alone is insufficient: a live scheduler can still fail to enqueue one tenant, so monitor both global progress and per-tenant age.

Make verification idempotent. A retry can overlap another worker or arrive after a tenant edits its requested domain. Bind each attempt to the current tenant and hostname version; discard results for an older version. Require a fresh matching DNS observation before marking the current version verified. Use bounded retries with backoff for transient resolver errors, and keep a manual retry path for support that goes through the same checks. Never interpret a timeout as a successful match.

## One verification loop that can recover

The example below describes application state rather than a particular queue or DNS library. The `dns` interface resolves the required challenge record through your configured verification path; storage must make `claimDue` exclusive and persist the claim across worker restarts. A separate recurring dispatcher invokes `runDue`, while the recovery sweep can invoke the same function. In production, cap claims per run and persist errors and attempt timestamps even when lookup fails.

```ts
type DomainClaim = {
  tenantId: string;
  hostname: string;
  version: number;
  expectedTxt: string;
};

type Store = {
  claimDue(now: Date): Promise<DomainClaim | null>;
  finish(claim: DomainClaim, result: {
    observed: string[] | null;
    error: string | null;
    checkedAt: Date;
  }): Promise<void>;
  recordCycleCompleted(at: Date): Promise<void>;
};

type Resolver = {
  resolveTxt(hostname: string): Promise<string[]>;
};

async function runDue(store: Store, dns: Resolver): Promise<void> {
  const claim = await store.claimDue(new Date());
  if (claim) {
    let observed: string[] | null = null;
    let error: string | null = null;
    try {
      observed = await dns.resolveTxt(claim.hostname);
    } catch (cause) {
      error = cause instanceof Error ? cause.message : String(cause);
    }
    await store.finish(claim, { observed, error, checkedAt: new Date() });
  }
  await store.recordCycleCompleted(new Date());
}
```

`finish` must compare the claim's version with the current hostname version in one transaction. Only an exact TXT match for that version may transition it to verified; otherwise it records the observation and schedules the next attempt. The scheduler heartbeat belongs after the work completes, not when a timer merely fires. One call processes one claim here to make the state boundary visible; a real dispatcher repeats with a bounded work budget and records failures so a crashed worker cannot silently strand its claim.

## When is the slower scan the right choice?

Use the batch-first approach when a delayed cutover is acceptable and the number of pending domains is small enough to recheck within a bounded run. It is easier to operate, which matters when the same person owns the release and the support inbox. Still store last-attempt times and alert on an overdue completed scan. A scan without that signal is a blind spot, however short its interval looks in configuration.

Before deploying either design, test a missing record, a wrong value, a resolver timeout, a stopped dispatcher, a worker restart after claim, and a hostname edit while an old attempt is in flight. Verify that an overdue tenant appears in an operational view and that a manual retry cannot bypass ownership checks. Track time from observed matching record to verified state separately from time waiting on customer DNS; otherwise a propagation chart can conceal a broken scheduler. Outsource undifferentiated queue operations if they consume feature time, but retain the tenant state and these alerts in your own system. The cutover promise depends on them.

## References

- RFC 1034, Domain Names: Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names: Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
