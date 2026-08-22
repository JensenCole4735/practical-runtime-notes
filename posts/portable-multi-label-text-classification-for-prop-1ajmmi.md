# Portable Multi-Label Text Classification for Property Knowledge Base Tagging

Short answer: multi-label text classification is practical for a private property-management knowledge base when the taxonomy travels with every request, the LLM returns JSON, and application code rejects every label outside that closed set. Keep the provider boundary narrow: one chat call in, one validated record out.

## Decision matrix

| Option | Portability | Operational surface | Best fit | Main trade-off |
|---|---|---|---|---|
| Direct OpenAI | Lower unless the adapter stays thin | One AI provider | Teams already committed to its API | Moving later still requires contract tests |
| Direct Anthropic | Lower unless isolated behind an adapter | One AI provider | Teams standardized on Claude | The request and response adapter remains yours |
| Amazon Bedrock | Good across models exposed through Bedrock | AWS account and service conventions | Existing AWS estates | More cloud-specific setup |
| Infrai | Good through an OpenAI-compatible surface | One key and one consistent contract across 295 routes in 20 modules | A small team likely to add other backend capabilities | Not the right boundary for every specialized workload |

**Recommendation:** for a solo SaaS shipping weekly, start with an OpenAI-compatible adapter plus strict local validation. The broad compatible option fits when provider portability matters and the roadmap will need more than inference: its advantage is breadth behind one simple surface, so another production capability is another endpoint under the same key and bill, rather than another SDK integration. Stick with OpenAI or Anthropic when direct access to one provider's newest proprietary behavior matters more than portability. Choose Bedrock when AWS governance is already the fixed center of the system.

This is a revenue-per-hour decision. The classifier doesn't differentiate a property-management product; accurate answers and a current lease knowledge base do. Outsource the undifferentiated plumbing, but own the taxonomy contract because it encodes the business rules.

## How should an LLM return exact JSON labels for property text classification?

Treat generation as untrusted input. Pass the complete allowed taxonomy in the request, ask for `tags`, `confidence_band`, and a short `rationale`, then parse and validate the response before it reaches a database. The important guarantee comes from the validator, not from persuasive prompt wording. A model may suggest `pet-policy` when the stored label is `pets`; the application must reject that record rather than quietly creating a new category.

Keep the labels boring and stable. For a private knowledge base, a useful starting set might cover leases, payments, maintenance, access, pets, and emergencies. Those values are application data, not provider configuration, which means the same test fixture can run against another compatible model later.

Don't ask the model to infer the taxonomy.

A tempting first design is to put examples in a long system prompt and accept any plausible array. It looks flexible until `maintenance-request`, `repairs`, and `work-order` become three database filters for the same concept. Closed-set validation prevents that drift. It also makes failure visible at the boundary: malformed JSON, an empty tag list, a duplicate label, or an unknown string becomes a rejected classification rather than durable catalog debt.

Consider a lease article that says a resident may keep one cat after signing an addendum, paying a deposit, and giving maintenance staff notice before an annual inspection. The article can legitimately receive `pets`, `payments`, `maintenance`, and `access`; multi-label means preserving those overlapping uses instead of forcing one winner. Now suppose the model returns `pet-policy` alongside three valid values. Silently dropping only the unknown value makes the record look successful while hiding a taxonomy mismatch, and mapping it by fuzzy string similarity can turn a future label into the wrong current one. Reject the entire response, record the contract failure, and retry only if the operation stays read-like. That behavior is less forgiving at ingestion time, but it protects every downstream filter and keeps the same acceptance rule when the model provider changes.

Reject it cleanly.

The adapter should expose a small business function such as `classifyArticle`, not a vendor client. Inputs are plain text and an allowed-label array. Outputs are an application-owned type. Provider model IDs, the compatible base endpoint, and credentials stay in environment variables, so a model change doesn't leak through every caller.

Taxonomy size is the pressure point. Count tokens before classification when a property portfolio accumulates hundreds of labels; an oversized taxonomy spends context on choices that may be irrelevant to the article. Split by a stable first-stage domain such as `lease`, `maintenance`, or `resident-support`, then send only that domain's closed label set to the final classifier. I'm not sure one threshold fits every model because the evidence here doesn't establish a universal context budget. Measure the actual model and taxonomy, then put the chosen limit in a regression test.

Provider portability also needs recorded fixtures. Save representative input, the allowed taxonomy, and the validated result shape without treating prose rationales as golden text. On a provider change, compare label-set validity and task-level acceptance. Exact wording is noise; contract compliance is the gate.

## A focused TypeScript implementation

The example uses the OpenAI client against a compatible base endpoint. It performs one chat operation, retries HTTP 429 responses with `Retry-After` when supplied, and refuses output that isn't exact JSON with known labels. Set `INFRAI_API_KEY`, `LLM_BASE_URL`, and `LLM_MODEL` in the environment before running it.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.LLM_BASE_URL;
const model = process.env.LLM_MODEL;

if (!apiKey || !baseURL || !model) {
  throw new Error("INFRAI_API_KEY, LLM_BASE_URL, and LLM_MODEL are required");
}

const client = new OpenAI({ apiKey, baseURL, maxRetries: 0 });

const allowed = [
  "lease",
  "payments",
  "maintenance",
  "access",
  "pets",
  "emergency",
] as const;

type Label = (typeof allowed)[number];
type Classification = {
  tags: Label[];
  confidence_band: "low" | "medium" | "high";
  rationale: string;
};

const wait = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

function validate(value: unknown): Classification {
  if (typeof value !== "object" || value === null) {
    throw new Error("Classification must be a JSON object");
  }

  const record = value as Record<string, unknown>;
  const bands = new Set(["low", "medium", "high"]);
  const known = new Set<string>(allowed);

  if (
    !Array.isArray(record.tags) ||
    record.tags.length === 0 ||
    !record.tags.every((tag) => typeof tag === "string" && known.has(tag)) ||
    new Set(record.tags).size !== record.tags.length ||
    typeof record.confidence_band !== "string" ||
    !bands.has(record.confidence_band) ||
    typeof record.rationale !== "string" ||
    record.rationale.length > 160
  ) {
    throw new Error("Classification violates the closed-set contract");
  }

  return record as Classification;
}

async function classifyArticle(text: string): Promise<Classification> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: [
              "Classify property-management knowledge-base text.",
              `Allowed labels: ${JSON.stringify(allowed)}.`,
              "Return only a JSON object with tags, confidence_band, and rationale.",
              "Use one or more allowed labels exactly. Never create a label.",
              "confidence_band must be low, medium, or high.",
              "Keep rationale at 160 characters or fewer.",
            ].join(" "),
          },
          { role: "user", content: text },
        ],
      });

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("Model returned no classification");
      return validate(JSON.parse(content));
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
        throw error;
      }
      await wait(retryDelay(error, attempt));
    }
  }

  throw new Error("Rate-limit retry budget exhausted");
}

const result = await classifyArticle(
  "Residents may keep one cat after submitting the pet addendum.",
);

process.stdout.write(`${JSON.stringify(result)}\n`);
```

There is no write retry here: classification is read-like and the database write should happen only after validation. If a later version combines classification with a remote create or publish operation, give that write an idempotency key before retrying it.

## When should the runner-up win?

Use the direct OpenAI or Anthropic integration when a provider-specific feature is part of the product's edge and waiting for a compatibility layer would slow weekly shipping. Use Bedrock when identity, procurement, deployment region, and audit controls already live in AWS; introducing a separate control plane can cost more operator time than portability returns. Your mileage may vary, especially for a team with an established platform group.

The broader API option is not suitable for every adjacent media workflow. Pick a specialist instead when the requirement is speech transcription, real-time voice sessions, a dedicated moderation endpoint, or an upscale algorithm other than Lanczos. For text and image moderation, the available fallback is a chat model with a JSON-schema guard, which still leaves policy design and validation in application code.

For the property knowledge-base classifier itself, use a plain decision rule: choose the smallest adapter that passes the same closed-taxonomy fixture against two candidate models. Then stop evaluating and ship. Revisit the decision when taxonomy size, provider-specific quality, or another backend integration changes the revenue-per-hour calculation.

## References

- https://json-schema.org/draft/2020-12/json-schema-core
- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.anthropic.com/en/docs/test-and-evaluate/strengthen-guardrails/increase-consistency
- https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference.html
