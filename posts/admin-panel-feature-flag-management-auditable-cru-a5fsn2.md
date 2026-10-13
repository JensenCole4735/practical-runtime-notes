# Admin Panel Feature Flag Management — Auditable CRUD for Delivery Failures

The useful feature flag management trade-off is not a richer admin panel versus a thinner one. It is attributable change history versus telemetry noise. For a fintech notification service, I would put CRUD actions in an audit log and delivery outcomes in metrics and traces.

| Choice | Signal quality | Operator cost | Best fit |
| --- | --- | --- | --- |
| Audited control plane + bounded metrics | High: intent and outcome stay separate | Moderate | Production flags that affect delivery |
| Database CRUD + request logs | Mixed: actions and incidental traffic blur together | Low at first | A small prototype with no on-call use |
| Git-managed declarations | High review signal, slower emergency action | Moderate | Infrequent, planned flag changes |

**TL;DR:** build list, create, update, toggle, and retire operations behind one server-side command boundary. Record actor, reason, previous state, next state, and a correlation ID for each accepted mutation. Emit delivery metrics with bounded labels such as channel, result, and flag key; never put customer, message, or request IDs in metric labels. The admin panel then answers who changed policy, while runtime telemetry answers whether delivery got worse.

That split is the recommendation. It gives a solo operator evidence without turning every dashboard into an archaeology project.

## How should an admin panel handle feature flag management CRUD?

A feature flag row looks harmless: key, enabled state, perhaps a targeting rule. The consequences are not harmless. A toggle can suppress payment reminders, move fraud alerts to a new provider path, or change which customers receive a fallback message. `UPDATE flags SET enabled = false` preserves the final state and discards the decision that produced it.

An HTTP access log does not repair that gap. It can show that a `PATCH` returned successfully, but usually cannot answer what changed, why it changed, or which prior value must be restored. Conversely, copying the complete before-and-after document into every application log makes routine reads and health checks compete with the few events that matter during an incident.

There are two timelines:

1. The **control timeline** contains operator intent: create, edit rules, toggle, and retire.
2. The **delivery timeline** contains observed effects: accepted, attempted, delivered, retried, or permanently failed notifications.

Keep both. Join them with a stable flag key and time window, not with an unbounded identifier embedded in every metric series. Prometheus specifically warns that each unique label combination creates a new time series and advises against high-cardinality labels such as user IDs and email addresses. A notification ID has the same dangerous shape.

No audit, no mutation.

This is the first decision criterion: can an operator connect a deliberate configuration change to an outcome without polluting the primary telemetry?

## Preserve intent, then measure outcomes

The admin command should validate the requested transition, commit the flag state and its audit record atomically, and return a revision. That revision prevents two browser tabs from silently overwriting each other. A request based on revision 18 must fail if revision 19 already exists; the UI can reload and show the competing change.

Deletion needs stricter semantics. Hard deletion destroys the key needed to interpret old delivery records. Prefer a retirement transition that disables evaluation, records the reason, and hides the flag from the default active list. A separate retention job may purge data later under an explicit policy. This is less convenient than a red trash button. It is far more legible at 2 a.m.

The second criterion is alert usefulness. Page on customer-impacting outcomes, not admin activity alone. A toggle is context for an alert, not proof of an incident. For example, an alert could evaluate the ratio of permanent failures to attempted deliveries over a suitable window, with a minimum traffic condition so that one failed message out of one attempt does not page the operator. Exact thresholds must come from the service's delivery objective and traffic distribution; there is no honest universal number.

Use bounded dimensions. `channel=email|sms|push`, `result=delivered|retryable_failure|permanent_failure`, and a controlled flag key can support diagnosis. `customer_id`, `notification_id`, error text, and correlation IDs belong in logs or traces. The distinction limits metric cardinality while preserving detailed evidence where point lookups are appropriate.

Quiet is valuable. Every false page spends the same scarce hour that could have shipped a feature.

I optimize for that hour.

## A small TypeScript command boundary

The UI should not write flag rows directly. It should call a server-side function that owns authorization, validation, concurrency, persistence, and audit behavior. The following domain-sized example omits framework plumbing on purpose; the same boundary can sit behind a server action or an internal API handler.

```ts
type FlagState = {
  key: string;
  enabled: boolean;
  status: "active" | "retired";
  revision: number;
};

type FlagCommand =
  | { kind: "toggle"; enabled: boolean }
  | { kind: "retire" };

type ChangeContext = {
  actorId: string;
  reason: string;
  correlationId: string;
  expectedRevision: number;
};

type AuditEntry = {
  flagKey: string;
  action: FlagCommand["kind"];
  actorId: string;
  reason: string;
  correlationId: string;
  before: FlagState;
  after: FlagState;
  occurredAt: string;
};

interface FlagRepository {
  transact<T>(work: (tx: FlagTransaction) => Promise<T>): Promise<T>;
}

interface FlagTransaction {
  getForUpdate(key: string): Promise<FlagState | null>;
  save(flag: FlagState): Promise<void>;
  appendAudit(entry: AuditEntry): Promise<void>;
}

export async function changeFlag(
  repository: FlagRepository,
  key: string,
  command: FlagCommand,
  context: ChangeContext,
): Promise<FlagState> {
  if (context.reason.trim().length < 8) {
    throw new Error("A specific change reason is required");
  }

  return repository.transact(async (tx) => {
    const before = await tx.getForUpdate(key);
    if (!before) throw new Error("Flag not found");
    if (before.revision !== context.expectedRevision) {
      throw new Error("Flag changed; reload before retrying");
    }

    const after: FlagState = command.kind === "retire"
      ? { ...before, enabled: false, status: "retired", revision: before.revision + 1 }
      : { ...before, enabled: command.enabled, revision: before.revision + 1 };

    await tx.save(after);
    await tx.appendAudit({
      flagKey: key,
      action: command.kind,
      actorId: context.actorId,
      reason: context.reason.trim(),
      correlationId: context.correlationId,
      before,
      after,
      occurredAt: new Date().toISOString(),
    });

    return after;
  });
}
```

Authentication is not authorization. The server must derive `actorId` from the authenticated session rather than accept it from the browser, then check that the actor may change this flag. The correlation ID can arrive from trusted request middleware, but it should be normalized and length-bounded before storage. The clock used for ordering should be assigned by the server or database.

List views also deserve care. Return active flags by default, offer an explicit retired filter, and show revision, last change time, and last actor. Do not calculate live delivery health by scanning the audit table during each page load. Query the metrics system for aggregated outcome health, or maintain a bounded summary designed for that view.

The code has one intentionally sharp edge: audit persistence is in the same transaction as state persistence. If the audit append fails, the mutation fails. For this control plane, missing evidence is not an acceptable partial success.

## Test the failure paths before polishing the panel

The highest-value tests are transition tests. Verify that a stale revision cannot overwrite a newer value, retirement always disables evaluation, a short or blank reason is rejected, an unauthorized actor cannot mutate state, and a failed audit append rolls back the flag update. Then test the visible workflow: the list refreshes after a successful toggle, conflict errors prompt a reload, and retired entries disappear from the active view without vanishing from history.

Deployment should keep schema and application versions compatible during rollout. Add audit storage before code starts requiring it. Deploy readers that understand the new status before writers can produce it. This two-step habit costs an extra release, but a weekly shipping rhythm can absorb that better than an emergency repair to an unreadable history.

A CI workflow can run type checks, transition tests, and migration checks on every change. GitHub Actions is one available automation mechanism, and its documentation describes workflows as configurable automated processes made of jobs. The architectural point is independent of the runner: the same checks must gate the admin service and its schema changes.

Do not stop at a green mutation test. Send synthetic delivery attempts through enabled, disabled, retryable-failure, and permanent-failure paths, then assert that the expected bounded metrics appear. Also inspect the label set. A test that rejects forbidden labels such as `customer_id` catches a costly observability mistake before production traffic multiplies it.

## When is the runner-up better?

Git-managed declarations are the runner-up when changes are planned, infrequent, and review latency is acceptable. A pull request naturally carries a diff, reviewers, and a commit history. It also outsources much of the approval interface to tooling the team already operates. For a one-person SaaS, that is appealing: less custom surface area means more revenue-producing hours remain available.

It is weaker for time-sensitive operational switches. Waiting for CI and deployment can be the wrong trade when a delivery route must be disabled promptly, and nontechnical operators may not have repository access. A hybrid can work if ownership is unambiguous: stable definitions and defaults live in version control, while narrowly scoped runtime overrides use the audited control plane. Define precedence and expiry rules once. Otherwise two sources of truth create more noise than either design alone.

Plain database CRUD plus request logging is reasonable only while the flag has no meaningful production consequence and no one expects an incident trail. Treat that as a temporary constraint, not an observability strategy. The moment a flag affects regulated or customer-critical notification behavior, state history and delivery evidence become part of the feature.

The practical choice is compact: preserve operator intent in an immutable audit stream, measure customer outcomes with bounded dimensions, and connect the two during investigation. Build the smallest panel that enforces those rules. Outsource the undifferentiated plumbing, keep the decision boundary yours, and get back to shipping.

## Sources

- Prometheus, "Instrumentation": https://prometheus.io/docs/practices/instrumentation/
- GitHub Docs, "GitHub Actions documentation": https://docs.github.com/en/actions
