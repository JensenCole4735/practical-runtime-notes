# Unified LLM API for Node.js: One-Key Invoice Correctness Across Providers

Short answer: put one gateway credential in the Node.js service, but do not treat a unified response shape as proof that an invoice was extracted correctly. For a one-person e-commerce SaaS, the least complex useful design is a small application-owned validation boundary behind the gateway. It accepts one internal request, rejects structurally invalid output, and sends uncertain invoices to review. The gateway handles provider access; the application owns the invoice contract.

| Choice | Operational burden | Correctness boundary | Best fit |
| --- | --- | --- | --- |
| Managed shared-key gateway | Low | Application validator plus gateway behavior | Shipping weekly with little infrastructure time |
| Self-hosted shared-key gateway | Medium to high | Application validator plus gateway deployment | Teams that need control over the gateway runtime |
| Direct provider adapters | Medium | Application validator plus three adapters | Provider-specific behavior is a product requirement |

**Recommendation:** start with the shared-key option whose data handling and deployment region meet your requirements, then keep schema validation and review policy in your own Node.js code. The runner-up is direct adapters, not because three SDKs are elegant, but because they preserve provider-specific controls when those controls matter more than a small integration surface.

## Should one unified LLM API key sit behind a Node.js backend?

One credential simplifies secret distribution, rotation, and the call site. It does not make OpenAI, Claude, and Gemini interchangeable. A normalized request can hide transport differences while output quality still varies with the model, prompt, document, and schema.

That distinction matters for invoices. A syntactically valid object can still assign the purchase-order number to `invoiceNumber`, turn a tax identifier into a total, or omit a currency that was visible on the page. JSON parsing catches none of those mistakes.

Keep the promise narrow: the gateway returns a candidate extraction. Your service decides whether that candidate is usable. This is the part tied to revenue per hour. My decision rule is blunt: I would rather ship a compact validator this week than spend the week recreating routing, credential handling, and retry plumbing that does not differentiate an invoice product.

## Correctness must survive normalization

Use a canonical domain type that is smaller than any provider's feature set. Every field should have a reason to exist in the order workflow. Avoid a generic `metadata` bag; it turns disagreements into silent production data.

The first layer checks shape. The second checks invoice semantics: totals cannot be negative, currency must use the format your system accepts, and line-item arithmetic must agree within a declared tolerance. The third layer handles uncertainty. Missing or contradictory fields go to review rather than being invented by a retry loop.

Three layers. No magic.

A model-generated confidence number is not evidence by itself. If confidence controls automation, calibrate it against labeled invoices from the supplier layouts you actually receive. Without that dataset, use deterministic validation failures and explicit review states.

A unified endpoint can make network errors look consistent. The business failures remain different: unreadable input, invalid JSON, schema mismatch, arithmetic conflict, missing required fields, or an extraction that passes syntax but fails human review. Preserve those categories in logs and metrics. The trade-off is explicit: a shared gateway buys a smaller operational surface, while direct adapters expose more provider-specific controls. Neither choice validates an invoice for you, and the common layer is not a fit when it hides a control that your acceptance policy requires.

Do not let a transport retry become an accounting policy. Retry timeouts and transient service failures with a capped attempt count, and apply backoff when a gateway returns 429. If a successful extraction triggers a write, carry an idempotency key derived from the invoice job so a retry cannot create a second record. Do not automatically retry a semantic mismatch against several models and accept the first answer that validates; that rewards agreement with the schema, not agreement with the invoice. Store the source document identifier, prompt version, schema version, selected model alias, validation result, and final review disposition. Avoid logging raw invoice content unless your retention and access rules explicitly allow it. A wrong payable total deserves a different queue and alert from a temporary network failure because the recovery action is different, and merging both into `failed` leaves the operator guessing at exactly the moment the automation was supposed to save time.

For US/EU operation, region is a deployment constraint rather than a label in a comparison table. Verify where requests, logs, and stored documents are processed for the exact service configuration you plan to use. Then test that configuration. A provider list alone cannot answer the data-flow question.

That boundary matters.

## A thin TypeScript boundary

The useful abstraction is not a universal model SDK. It is one function that returns either a validated invoice or a typed failure. The gateway URL, credential, and model alias remain configuration, while invoice rules stay in the application.

```ts
type Invoice = {
  invoiceNumber: string;
  supplierName: string;
  currency: string;
  total: number;
};

type ExtractionResult =
  | { ok: true; invoice: Invoice }
  | { ok: false; reason: "transport" | "invalid_json" | "invalid_invoice" };

function isInvoice(value: unknown): value is Invoice {
  if (typeof value !== "object" || value === null) return false;
  const row = value as Record<string, unknown>;

  return (
    typeof row.invoiceNumber === "string" &&
    row.invoiceNumber.length > 0 &&
    typeof row.supplierName === "string" &&
    row.supplierName.length > 0 &&
    typeof row.currency === "string" &&
    /^[A-Z]{3}$/.test(row.currency) &&
    typeof row.total === "number" &&
    Number.isFinite(row.total) &&
    row.total >= 0
  );
}

async function extractInvoice(documentText: string): Promise<ExtractionResult> {
  let response: Response;

  try {
    response = await fetch(process.env.LLM_GATEWAY_URL!, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.LLM_GATEWAY_KEY}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        model: process.env.INVOICE_MODEL_ALIAS,
        input: documentText,
        schema: {
          invoiceNumber: "string",
          supplierName: "string",
          currency: "three uppercase letters",
          total: "non-negative number",
        },
      }),
    });
  } catch {
    return { ok: false, reason: "transport" };
  }

  if (!response.ok) return { ok: false, reason: "transport" };

  let candidate: unknown;
  try {
    candidate = await response.json();
  } catch {
    return { ok: false, reason: "invalid_json" };
  }

  if (!isInvoice(candidate)) {
    return { ok: false, reason: "invalid_invoice" };
  }

  return { ok: true, invoice: candidate };
}
```

This example deliberately stops before persistence. In production, add the arithmetic and supplier-specific checks before an `ok` result can update an order. Also replace the illustrative request contract with the contract documented by the gateway you select; there is no standard universal LLM request route implied here.

Test the boundary with a fixed corpus that includes clean invoices, rotated scans, duplicate invoice numbers, missing currencies, tax-inclusive totals, and adversarial text inside the document. Record field-level outcomes. A single whole-document pass rate hides the difference between a harmless missing address and a wrong payable total.

## When is the runner-up better?

Choose direct provider adapters when your product depends on provider-specific structured-output controls, document inputs, regional configurations, or error details that a common layer cannot expose without loss. It is also the clearer choice when procurement requires separate credentials and audit trails. You pay for that control with more integration and operational work.

Self-hosting the shared layer is reasonable when runtime control is mandatory and you can own upgrades, observability, and availability. LiteLLM is an open-source example of this gateway category. Its existence proves that consolidation does not require a managed intermediary; it does not remove the work of operating the intermediary.

Do not select from a logo grid. Run the same labeled invoice corpus through the exact models and configurations under consideration, then inspect field correctness, invalid-output handling, regional data flow, and recovery behavior. Price belongs in the final decision, but a cheap wrong total is still wrong.

The durable choice is the one that leaves invoice correctness visible in your code and operations. Outsource undifferentiated provider plumbing when its boundaries fit. Keep the business verdict close.

Ship the boundary first.

## Further reading

- https://github.com/BerriAI/litellm
- https://platform.openai.com/docs/guides/embeddings
