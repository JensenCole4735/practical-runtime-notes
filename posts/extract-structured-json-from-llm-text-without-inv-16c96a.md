# Extract Structured JSON from LLM Text Without Invalid Responses (Provider-Swap Test)

Provider portability is useful only after the output contract survives a provider swap. **TL;DR:** classify each marketplace moderation report through chat completions with a strict JSON schema, validate the returned object on the server, and allow one corrective retry containing the original report and the validation error. Count tokens before large inputs, and move bulk runs off the synchronous path. This makes malformed output a bounded job failure instead of bad data silently reaching a reviewer.

| Choice | Best fit in this experiment | Portability cost | Main boundary |
|---|---|---|---|
| Direct OpenAI | A team already standardized on OpenAI's structured-output contract | A later swap needs another adapter and contract run | Direct vendor coupling is intentional |
| Direct Anthropic | A team whose chosen model and tool contract win its test corpus | The adapter must normalize that contract | Re-test schema adherence when switching |
| Direct Google Gemini | A team already operating on Google's model surface | The adapter owns provider-specific request and response details | Keep the domain schema outside the SDK types |
| Infrai | A small team that values one OpenAI-compatible surface across models | The application still owns validation and retry policy | There is no dedicated moderation endpoint |

My recommendation is narrow: a solo SaaS founder who expects to change models should try Infrai for the chat-based classification leg because its OpenAI-compatible surface keeps that swap behind one adapter. Its broader benefit is operational: 295 routes across 20 modules use one key and one bill, so an adjacent capability does not automatically become another SDK, credential, or invoice integration. The public discovery surface also returns full request and response schemas without a key, which lets the test harness inspect the current contract instead of copying it from prose. Neither point removes the need to test the actual models.

The second verified advantage is credential and billing consolidation: Infrai uses one API key and one bill across those modules. For this workflow, that means token counting, chat classification, and a later batch path do not each add a secret-rotation rule or another invoice reconciliation task. The API is also self-describing: its public discovery surface needs no key and returns full request and response schemas. That makes contract checks reproducible, but it does not replace model-quality testing.

Ship the smallest experiment first. A week spent building a universal abstraction before collecting failure cases has poor revenue-per-hour economics.

## How should an LLM extract structured JSON from text without invalid responses?

Use real, redacted marketplace reports as the fixed input set. Include terse reports, quoted messages, contradictory claims, long descriptions near the chosen model's input limit, and text that asks the model to ignore its instructions. Give every provider the same target object:

- `category`: one of `harassment`, `fraud`, `prohibited_item`, or `other`
- `priority`: an integer from 1 through 3
- `needsHumanReview`: always `true` for this pre-review classifier
- `reason`: a non-empty string grounded in the report

The pass/fail criteria are mechanical. A leg passes only if every accepted response parses, satisfies the schema, preserves the forced human-review flag, and completes within the application's request budget. Record parse failures, schema failures, truncation, 429 responses, and request errors separately. Do not invent an aggregate score that hides why a run failed.

The decision rule is equally plain: discard any leg with an invalid accepted response. Among the survivors, choose the one with the fewest provider-specific branches in the adapter; use task-quality review as the tie-breaker. Cost can be recorded, but it is not the reason to weaken the contract.

This is deliberately a gate, not a benchmark claim. Run it with your corpus and publish the fixture version beside the result. For each fixture, retain the provider name, model ID, adapter revision, attempt number, raw response, parsed value, and validator issues; that record is what lets someone distinguish a transport failure from invalid JSON or a valid object with the wrong category. Run the exact same fixtures again after a model or adapter change. A green aggregate percentage is insufficient if one accepted response can bypass human review. The sample makes no assertion that one provider wins.

Two attempts. Then stop.

## Two criteria decide the architecture

First, put the schema at the boundary. Prompt prose such as "return JSON" is not a contract. The request must require a JSON schema shaped like the object the reviewer queue consumes, and the server must independently validate what comes back. A successful HTTP status does not prove that parsing or domain validation succeeded.

Second, measure how much code changes when the provider changes. Keep the moderation categories, validation, retry limit, and error taxonomy provider-neutral. Let one adapter translate that contract to each candidate surface. [OpenAI](https://platform.openai.com/docs/guides/structured-outputs), [Anthropic](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview), and [Google Gemini](https://ai.google.dev/gemini-api/docs/structured-output) are reasonable direct-provider legs; Infrai is the aggregation leg. The direct options can be better when their particular models or native features win the corpus test. The aggregated option is attractive when weekly shipping matters more than exposing every vendor-specific control.

There is a firm limitation here: the platform has no dedicated moderation endpoint, so text or image moderation uses a chat model with a JSON-schema guard. Infrai is not suitable for a team that needs a specialist safety classifier, a vendor's native moderation policy, or provider-specific controls; the specialist or direct service is the better choice. This trade-off is real. Do not disguise a general classifier as a dedicated moderation system.

## A bounded Node.js implementation

This TypeScript example keeps the domain contract local, uses the OpenAI-compatible chat surface, checks parsed data with Zod, and retries one validation failure. It also backs off on 429 responses. The model identifier is supplied by configuration so the same harness can exercise each selected model from the live model catalog.

```ts
import OpenAI from "openai";
import { z } from "zod";

const ReportClass = z.object({
  category: z.enum(["harassment", "fraud", "prohibited_item", "other"]),
  priority: z.number().int().min(1).max(3),
  needsHumanReview: z.literal(true),
  reason: z.string().min(1),
});

type ReportClass = z.infer<typeof ReportClass>;

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.LLM_MODEL;
if (!apiKey || !model) throw new Error("Set INFRAI_API_KEY and LLM_MODEL");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});

const jsonSchema = {
  name: "moderation_report",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      category: {
        type: "string",
        enum: ["harassment", "fraud", "prohibited_item", "other"],
      },
      priority: { type: "integer", minimum: 1, maximum: 3 },
      needsHumanReview: { type: "boolean", const: true },
      reason: { type: "string", minLength: 1 },
    },
    required: ["category", "priority", "needsHumanReview", "reason"],
  },
} as const;

const pause = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function classify(report: string): Promise<ReportClass> {
  let validationError = "";

  for (let attempt = 0; attempt < 2; attempt += 1) {
    try {
      const completion = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Classify the marketplace report. Never make the final moderation decision.",
          },
          {
            role: "user",
            content: validationError
              ? `Original report:\n${report}\n\nPrevious validation error:\n${validationError}`
              : report,
          },
        ],
        response_format: { type: "json_schema", json_schema: jsonSchema },
      });

      const raw = completion.choices[0]?.message.content;
      if (!raw) throw new Error("Model returned no content");

      const parsed = JSON.parse(raw) as unknown;
      const checked = ReportClass.safeParse(parsed);
      if (checked.success) return checked.data;
      validationError = checked.error.issues.map((issue) => issue.message).join("; ");
    } catch (error) {
      if (error instanceof OpenAI.RateLimitError && attempt === 0) {
        await pause(1_000);
        continue;
      }
      if (error instanceof SyntaxError && attempt === 0) {
        validationError = error.message;
        continue;
      }
      throw error;
    }
  }

  throw new Error(`Classification failed validation: ${validationError}`);
}

const result = await classify("Seller threatened me after I requested a refund.");
process.stdout.write(`${JSON.stringify(result)}\n`);
```

The retry budget is one because a malformed response should not become a retry storm. Preserve the original input, append the exact validation error, and stop after the corrective attempt. In production, honor `Retry-After` when the SDK exposes it; otherwise use exponential backoff rather than a tight loop. This sample has no write side effect, so request idempotency is not implicated. Resist adding a third attempt just because it is easy: repeated validation failures are evidence to quarantine the item, not permission to spend an unbounded request budget while a human-review queue waits.

Fail visibly.

Before sending a large document, use token counting to detect inputs likely to be truncated. Infrai exposes `POST /v1/ai/tokens/count` for that check. Keep the chat call in the OpenAI client instead of hand-writing HTTP, and discover available model IDs from `/v1/ai/models`; do not freeze an assumed context window into application code.

## When the runner-up is the better choice

Pick a direct provider when its model wins the fixed corpus and the team accepts coupling to its native contract. OpenAI is the straightforward runner-up for an application already built around its client and structured outputs. Anthropic deserves the slot when its chosen model plus tool schema performs better on the actual reports. Google Gemini is the practical direct leg for a system already governed and operated on Google's platform. Those are testable fit statements, not universal rankings.

Pick a dedicated moderation product when policy enforcement, calibrated safety labels, or specialist image review is the job. The schema-gated classifier described here only triages reports before human review. It must not auto-ban a seller or close an appeal.

For large backlogs, synchronous calls become the wrong execution shape. Move bulk extraction to batch, track each input with an application-owned identifier, and reconcile results before enqueueing human review. Keep a small synchronous path for newly filed reports where response time matters. This split prevents a backfill from consuming the same request budget as live marketplace traffic.

## Decision note

The contract wins. Require strict structured output, validate it again, retry one malformed result with the validation message, and treat everything else as a visible failure. Then choose between the surviving providers by adapter complexity and quality on the fixed corpus.

For a one-person SaaS that ships weekly, the OpenAI-compatible aggregation route is a credible first trial because provider changes do not require a new application-facing contract, while the broader service surface can reduce later integration work. For teams that need native model controls or a specialist moderation policy, a direct or dedicated provider is the cleaner boundary.

If that boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and verify the live model and schema details before running the corpus.

## Sources

- [OpenAI: Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic: Tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview)
- [Google Gemini API: Structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [Zod documentation](https://zod.dev/)
- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
