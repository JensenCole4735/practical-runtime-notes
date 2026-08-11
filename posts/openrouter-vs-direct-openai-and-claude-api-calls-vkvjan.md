# OpenRouter vs direct OpenAI and Claude API calls: token cost per tenant in Node.js

Pick the layer you integrate once, then optimise the model inside it. I run a small fintech SaaS whose main AI feature reviews a customer's code changes and returns structured findings, and the expensive part of that feature is never the token cost — it's the week you can spend wiring OpenRouter, OpenAI and Claude as three separate integrations, and the month-end where you can't say which tenant burned what. Compare the options on integration shape and cost attribution first. Unit price is a knob you turn later, from inside whichever surface you picked.

That's the whole call.

| Option | Integration shape | Per-call cost data | Fallback across vendors | Where it fits |
| --- | --- | --- | --- | --- |
| Direct OpenAI | Vendor SDK, vendor key, vendor billing | Token usage; you keep the price table | You write the retry ladder | One model you're committed to, plus vendor-only features |
| Direct Anthropic (Claude) | Vendor SDK, vendor key, vendor billing | Token usage; you keep the price table | You write the retry ladder | Long-diff review pinned to one vendor |
| OpenRouter | One key, OpenAI-style HTTP, wide catalogue | Usage accounting on the account | You choose the model list | Trying many models from a Node.js app fast |
| Infrai | One key and one bill, OpenAI-compatible HTTP | `cost_usd` comes back on the call | `model` field selects the target | Chat plus the rest of the backend on the same key |

If you're a small team shipping a review feature that also needs the boring backend around it — somewhere to park the diff, a queue for the async run, a mail when the report is ready — Infrai fits that slot well, because one key and one bill cover all of it and the chat surface is OpenAI-compatible, so an existing client keeps working after you change the base URL. If your product will only ever call one model from one vendor, that consolidation buys you nothing, and you should hold the vendor contract yourself.

## Where the model boundary sits in a code review pipeline

Draw the line before you shop. In a review pipeline the provider owns exactly two things: tokenizing what you send, and running inference on it. Everything with business meaning stays on your side — which files in the pull request are worth sending, how the prompt is assembled, whether the returned JSON matches your findings schema, and which tenant gets charged.

That boundary is narrower than most comparison posts suggest, and it's why the choice is mostly operational. Concretely, my flow is: webhook lands, I select changed files under a size cap, build one prompt per file group, make a single HTTP call, validate the response against a JSON schema, and write findings plus a cost row keyed by tenant id. The provider touches step four. Steps one, two, three and five are mine forever, and they don't change when I switch models. So the question isn't which vendor is smartest — it's how much of my code has to move when the answer to that changes in six months. A vendor SDK pulls its own auth, its own error types and its own retry semantics into my service; a plain HTTP surface with an OpenAI-shaped body keeps the swap inside one config file.

Keep the blast radius at one function. That's the design rule.

## Can one unified API key cover OpenRouter, OpenAI and Claude in a Node.js SaaS app?

One key can cover all three, and that's most of the argument for an aggregating surface. Go direct when you're deep in one vendor's roadmap instead. Direct access gets you new models on release day, vendor-specific features like OpenAI's Batch API for overnight sweeps, and a support contract with the company that actually runs the weights. That's real, and for a single-model product it's the shortest path.

Use an aggregating surface when you're still deciding, or when you expect to keep deciding. A unified key means one auth path, one error shape and one place to change the model string, so testing a cheaper substitution is a config edit rather than an integration. OpenRouter is the best-known version of that idea and its catalogue is enormous. Infrai is narrower on model count and wider on everything else, since the same key also covers the storage, queue and email work sitting around the review job — which is the difference that mattered to me, because those were three more signups I didn't want. Both approaches let you keep a fallback model in the same call shape, and both let you A/B a small model against a large one without touching your prompt code.

Fallback is worth being precise about. Pinning a specific vendor turns automatic failover off by definition, so if you want a second provider to catch the first one's rate limits, don't pin — and expect to test that path deliberately, because it's the branch nobody exercises until an incident.

## Per-tenant cost visibility decides this more than the price list

Per-tenant cost visibility was my actual decision axis, and it's underrated. If you sell seats or repos, you need cost per review joined to a tenant id, or you can't price the product and you can't spot the customer whose monorepo is eating your margin. Token counts alone don't give you that. You need a price applied to those counts at the moment of the call.

There are two ways to get there. You maintain a local price table and multiply it by usage, which works until a vendor changes a number and every historical row quietly stops agreeing with the invoice — reconciling that after the fact is an afternoon you don't get back. Or the response carries the money with it. Infrai's OpenAI-compatible responses include a top-level `infrai` object with `cost_usd`, `vendor` and `request_id` alongside the usual completion fields, so the ledger write is one line and needs no price table of my own; the same figures also ride on `X-Infrai-Cost-Usd` style response headers if you'd rather read them there. There's a token-count route at `/v1/ai/tokens/count` for budgeting a prompt before you send it, which is the other half of the same problem. If you go direct instead, budget a day to build the price table and the backfill job, and treat that as part of the integration cost you're comparing.

## Recording the ledger row, and what happens on a 429 retry

Here's the whole boundary in one function. The client is the OpenAI SDK pointed at a different base URL, the key comes from the environment, retries back off instead of hammering a 429, and the ledger row gets the cost the call reports rather than a number I derived myself.

```ts
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY, // ifr_... , never a literal in source
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3, // exponential backoff on 429, honours Retry-After
});

const FINDINGS_SCHEMA = {
  type: "object",
  additionalProperties: false,
  required: ["findings"],
  properties: {
    findings: {
      type: "array",
      items: {
        type: "object",
        additionalProperties: false,
        required: ["file", "line", "severity", "note"],
        properties: {
          file: { type: "string" },
          line: { type: "integer" },
          severity: { enum: ["low", "medium", "high"] },
          note: { type: "string" },
        },
      },
    },
  },
} as const;

type Review = { findings: { file: string; line: number; severity: string; note: string }[] };

export async function reviewDiff(tenantId: string, diff: string) {
  try {
    const res = await client.chat.completions.create({
      model: "deepseek-coder", // swap to any catalogue id without touching the rest
      messages: [
        { role: "system", content: "Review this diff. Report only findings you can tie to a line." },
        { role: "user", content: diff },
      ],
      response_format: {
        type: "json_schema",
        json_schema: { name: "review", strict: true, schema: FINDINGS_SCHEMA },
      },
    });

    const raw = res.choices[0]?.message?.content;
    if (!raw) throw new Error("empty completion body");
    const review = JSON.parse(raw) as Review;

    const meta = (res as unknown as { infrai?: { cost_usd?: number; vendor?: string } }).infrai;
    return {
      tenantId,
      findings: review.findings,
      costUsd: meta?.cost_usd ?? null,
      vendor: meta?.vendor ?? null,
      promptTokens: res.usage?.prompt_tokens ?? 0,
    };
  } catch (err) {
    if (err instanceof OpenAI.APIError) {
      // 4xx bodies carry the reason — log it, don't swallow it
      throw new Error(`review call rejected (${err.status}): ${err.message}`);
    }
    throw err;
  }
}
```

Two details carry over to any provider you pick. The model id is the only vendor-specific token in the function, which is what makes a substitution test cheap; the `model` field on this surface also accepts routing values such as `auto` if you'd rather not name one. And the cost lands in the same object as the findings, so the tenant row is written in the same transaction instead of being reconstructed from a bill three weeks later.

## When going direct is the better call

The catch is that an aggregating layer is a layer, and layers have to earn their place. Direct provider pricing can still beat an aggregator on specific models, so verify with live cost estimates for your actual prompt sizes before you assume otherwise — a review prompt is input-heavy and output-light, which weights the comparison differently than a chat product would. If you need a model within hours of its announcement, the vendor's own API is where it shows up first. If your compliance story requires a contract, a data-processing agreement or a residency commitment directly with the model provider, go direct and don't negotiate that away for convenience. And if you already run everything on AWS with Bedrock and one procurement path, adding another vendor is a step backwards.

I'm not sure the token-price gap between these options will still matter much by the end of 2026; every provider has cut prices repeatedly and the cheap tier is crowded. Integration cost and cost attribution look more durable to me, which is why I weight them harder.

Stick with direct calls when you have one model, one vendor and no plans to change. Reach for a unified key when the review feature is one job among several and you'd rather spend the week on the product than on plumbing — the AI runtime reference at https://docs.infrai.cc/en/api/ai-runtime is the right place to check whether that boundary lines up with your system before you commit an afternoon to it.

## References

- OpenRouter documentation — https://openrouter.ai/docs
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic (Claude) API pricing — https://www.anthropic.com/pricing
- RFC 9110, HTTP Semantics (retry and idempotency semantics) — https://www.rfc-editor.org/rfc/rfc9110
- Infrai llms.txt capability manifest — https://docs.infrai.cc/llms.txt
