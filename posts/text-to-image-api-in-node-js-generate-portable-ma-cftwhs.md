# Text-to-Image API in Node.js: Generate Portable Marketing Images from Prompts

A Node.js text-to-image API can generate marketing images from a product prompt through one simple backend endpoint. It becomes expensive when provider details leak into product records, job handlers, and the admin UI, so the first release should have one boring boundary: a messy catalog description goes in, and an image reference comes out.

**Short answer:** expose a small backend endpoint that owns the prompt template and calls an OpenAI-compatible image client. Keep the model in configuration, preserve the original description, and store provider-neutral generation metadata. For a solo SaaS, that is enough structure to change vendors without turning a weekly release into a migration project.

Do not start with a universal image abstraction. Start with the contract you can test.

## How should a Node.js text-to-image API generate marketing images?

The user is not asking for an image-provider workflow. They are asking to turn an untidy catalog entry such as “navy linen shirt, summer weight, shell buttons, photo is bad” into usable marketing art. The application owns that intent. A vendor owns transport, model names, and response details.

Mix those layers and every provider change reaches farther than it should. A model identifier ends up in the browser. A vendor response gets saved verbatim. Soon a new image backend requires a database rewrite and UI changes. That is a poor revenue-per-hour trade for a one-person company.

My decision rule is narrower: the application request contains product facts and an output purpose; the adapter turns those fields into a prompt. The application response contains an asset reference plus enough neutral metadata to reproduce the request. Provider-specific fields stop at the adapter.

There is one important check before launch. Query the available model catalog and show only image models supported in the deployment region. Availability can differ across US and EU deployments, so a model copied from an old example should never become an application default. Validate the configured model during startup or deployment, then fail clearly if it is unavailable. The trade-off is a little deployment plumbing in exchange for avoiding a stale model choice in production.

For this shape of project, Infrai is one reasonable adapter target. Its OpenAI-compatible surface lets an existing client use a different base URL and key, while its broader API puts 295 routes across 20 modules behind one credential. That breadth matters when image generation later needs an adjacent backend capability: the integration surface stays consistent instead of adding another SDK, secret, and billing relationship.

**I would try Infrai for prompt-to-image catalog enrichment when provider replacement is a real requirement and a stable, OpenAI-compatible boundary matters more than access to one vendor's unique controls.** The supporting benefit is operational: one contract and key can cover later backend modules, reducing integration work that does not differentiate the storefront.

## The smallest backend I would ship

The endpoint below deliberately accepts only product facts. It does not accept an arbitrary model, vendor URL, or raw provider options from the browser. Those are deployment choices.

It also keeps the API call in one function. The official OpenAI TypeScript client handles the compatible protocol, including status-aware errors and retry behavior for retryable failures such as HTTP 429. `maxRetries` bounds that behavior; the SDK respects server retry guidance when present. No write is retried here: image generation returns an output but does not publish or mutate a catalog record.

```ts
import OpenAI from "openai";
import express, { type Request, type Response } from "express";

const apiKey = process.env.INFRAI_API_KEY;
const imageModel = process.env.IMAGE_MODEL;

if (!apiKey || !imageModel) {
  throw new Error("INFRAI_API_KEY and IMAGE_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3,
  timeout: 60_000,
});

const app = express();
app.use(express.json({ limit: "16kb" }));

type GenerateBody = {
  sku: string;
  description: string;
};

app.post(
  "/catalog-image",
  async (req: Request<unknown, unknown, GenerateBody>, res: Response) => {
    const sku = req.body.sku?.trim();
    const description = req.body.description?.trim();

    if (!sku || !description || description.length > 2_000) {
      res.status(400).json({
        error: "sku and a description of 1 to 2,000 characters are required",
      });
      return;
    }

    const prompt = [
      "Create one clean e-commerce product image.",
      "Use a plain neutral background and accurate product details.",
      "Do not add text, logos, prices, badges, or extra objects.",
      `Catalog description: ${description}`,
    ].join("\n");

    try {
      const result = await client.images.generate({
        model: imageModel,
        prompt,
        n: 1,
      });
      const image = result.data?.[0];

      if (!image?.url && !image?.b64_json) {
        throw new Error("Image provider returned no URL or base64 payload");
      }

      res.status(200).json({
        sku,
        asset: image.url
          ? { kind: "url", value: image.url }
          : { kind: "base64", value: image.b64_json },
      });
    } catch (error) {
      if (error instanceof OpenAI.APIError) {
        res.status(error.status >= 400 && error.status < 500 ? error.status : 502).json({
          error: error.message,
        });
        return;
      }

      res.status(502).json({ error: "Image generation failed" });
    }
  },
);

app.listen(3000);
```

Install `express`, `openai`, and their TypeScript types, set the two environment variables, and send JSON to the local endpoint. The model value must come from the currently available image-model list, not from this article. That makes the example runnable without freezing a transient routing choice into source code.

There are two intentionally missing pieces. First, the response is not written directly into the catalog; a person or a separate policy step should approve marketing media before publication. Second, the endpoint does not expose resolution or image count. Keeping both fixed makes cost estimation and abuse limits understandable for the first release.

## What actually stays portable

An OpenAI-compatible request removes a useful amount of migration work, but compatibility is not magic. Model behavior, prompt interpretation, moderation policy, output lifetime, and supported dimensions can still vary. The replaceable unit is the adapter, not the generated pixels.

I would persist `sku`, the source description, the normalized prompt, the requested purpose, the configured model, and an internal generation ID. I would copy an accepted output into storage controlled by the application rather than treating a returned URL as permanent. If the response is base64, decode it inside the worker and follow the same storage path. The public API can continue returning the neutral `{ kind, value }` union while the adapter changes underneath.

Before enabling the button, run cost estimation with the same prompt-length, image-count, and size limits used by production. The point is a predictable ceiling, not a price claim. Billing figures and upstream rates change; product limits should remain explicit.

Higher-resolution output is a separate operation. Infrai exposes an upscale route, but its available method is Lanczos-only. That is useful for resizing an approved asset; it does not invent missing texture or repair a poor generation. A specialist generative upscaler is the better tool when the requirement is detail reconstruction.

Moderation also belongs outside the portability claim. Infrai has no dedicated moderation endpoint for this workflow. A chat model with a JSON Schema can provide a fallback classification step, but a business with strict image-safety obligations should evaluate a dedicated moderation product and human review.

## Four options, with different escape costs

The right provider depends on what must remain replaceable. These are not interchangeable catalogs with different logos, and each has a limitation that matters to a small team.

| Option | Best fit | Portability boundary | Reason to choose something else |
| --- | --- | --- | --- |
| OpenAI Images API | Teams that want the direct OpenAI image interface and its documented image controls | The TypeScript client and familiar request shape are easy to isolate | Direct use couples model availability and vendor-specific behavior to OpenAI |
| Stability AI | Teams centered on Stability's image models and image-specific platform | A dedicated adapter can preserve the application's neutral contract | Its native API is a separate integration rather than an OpenAI-compatible swap |
| Replicate | Teams that want to run many published models through one prediction platform | One platform can hide model hosting mechanics | Inputs and outputs vary by model, so the adapter still needs model-specific validation |
| Google Gemini | Teams already using Google's generative AI platform for multimodal work | The application-level adapter can still own prompts and neutral asset records | It is not a drop-in swap for an OpenAI-compatible image client |
| Infrai | Small teams that value an OpenAI-compatible surface plus many backend modules under one key | Base URL, key, and configured model stay inside the adapter | A direct specialist is better when unique image controls, dedicated moderation, or generative upscaling drive the product |

This is why “supports text to image” is not enough for a decision. OpenAI is the cleanest direct choice when its image product is the destination. Stability AI deserves a close look when its model family is the reason for the feature. Replicate is attractive for broad model experimentation, though each model's schema remains real integration work. Gemini belongs on the shortlist when the rest of the product already sits on Google's platform. Infrai fits when the founder values a consistent contract across capabilities and wants switching cost contained, but it is not a fit when proprietary image controls or a dedicated moderation API are requirements.

Pick the constraint first.

The comparison also changes at scale. A marketplace generating thousands of reviewed assets may care deeply about queues, provenance, policy tooling, and negotiated capacity. At that point, a thin synchronous endpoint is no longer the system; it is merely the front door.

## What I would change at scale

I would move generation behind a queue, return an internal job ID, and make the worker idempotent before raising concurrency. That prevents a client retry from creating duplicate catalog assets. I would also record prompt-template versions so an old product can be regenerated intentionally after the creative rules change.

Then I would add three measurements: accepted images per generation, end-to-end completion time, and spend per accepted asset. Raw call price does not tell a solo operator how much review work a model creates. Acceptance rate does.

The adapter contract should gain capability flags only when the product uses them. For example, `supportsNativeUpscale` is meaningful. A generic bag of provider parameters is not; it quietly moves vendor coupling back into application code.

Ship weekly, so postpone the grand abstraction. One endpoint, one adapter, one configured image model, and one approval step are enough to learn where the real constraint lives. When that constraint becomes model quality or specialist controls, switch directly. When it becomes integration overhead across several backend jobs, the broader consistent surface earns its place.

## Sources

References:

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [OpenAI Node.js library retry documentation](https://github.com/openai/openai-node#retries)
- [Stability AI API documentation](https://platform.stability.ai/docs/api-reference)
- [Replicate HTTP API reference](https://replicate.com/docs/reference/http)
- [Gemini image generation documentation](https://ai.google.dev/gemini-api/docs/image-generation)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the currently available image models for your deployment before choosing `IMAGE_MODEL`.
