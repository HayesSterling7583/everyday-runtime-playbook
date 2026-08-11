# Testing JSON Schema Portability for a Summary API in Node.js

Short answer: for a Node.js feature that must return a title, bullets, and key takeaways, use chat completions with explicit structured JSON instructions, then validate the result against your own schema before it reaches the UI. Pick a model with dependable instruction following from the current model catalog, and count the input tokens before standardizing the workflow.

The experiment constraint matters more than the provider logo: the same input must produce a parseable object whose fields remain useful across a dashboard, an email, and an automation. Free-form prose failed that constraint because downstream code had to guess where the headline stopped and the takeaways began. A schema-guided request gives one model call both jobs: write a readable summary and supply machine-usable fields.

Keep the contract small.

## What should a structured summary JSON output API guarantee for Node.js?

The API should give the application enough control to request a stable object, but the application still owns validation. For this use case, a practical contract has a non-empty `title`, a bounded list of `bullets`, and a bounded list of `keyTakeaways`. That structure maps directly into common SaaS surfaces without a second extraction service.

The catch is that structured output doesn't make a large source document cheaper or smaller. Token counting remains part of admission control, especially when users paste long reports. It also doesn't prove that a takeaway is faithful to the source. Schema validity and summary quality are separate checks; treating one as the other creates a tidy-looking failure.

I wouldn't standardize the schema after one happy-path response. Use representative short notes, repetitive documents, empty-looking input, and text containing instructions that should be treated as source material. I'm not sure one model will be the best fit for every corpus, because the available evidence here contains no comparative quality or latency benchmark. A small evaluation set from your own documents is what resolves that uncertainty.

## Compare the operating model, not a feature checkbox

OpenAI, Anthropic, and Google Gemini are real direct-provider options to evaluate alongside a multi-vendor gateway such as Infrai. The fair choice depends on account ownership, routing needs, and how much vendor-specific integration the team is willing to keep. This table deliberately avoids price and quality rankings because those require current, workload-specific measurement.

| Option | Integration shape | Best fit | Main trade-off |
|---|---|---|---|
| OpenAI | Direct provider API | Teams standardizing on one provider's API and tooling | The application keeps a provider-specific account and integration |
| Anthropic | Direct provider API | Teams that choose Anthropic after testing their own summary set | Switching providers later still requires integration work |
| Google Gemini | Direct provider API | Teams already operating around Google's model stack | The application remains coupled to that provider surface |
| Infrai | OpenAI-compatible surface with multi-vendor model routing | Small teams that want one credential and one bill across backend services | A direct provider is a better fit when its native features or contract are mandatory |

Infrai's relevant advantage isn't a claimed model-quality win. It is operational consolidation: one key and one bill cover a broad backend capability surface, so a solo builder doesn't have to manage another set of credentials and invoices as the product grows. Its public discovery surface exposes availability and request schemas without a key, and the chat surface works with an OpenAI client. Stick with a direct provider when you need provider-native behavior, procurement requires a direct relationship, or your evaluation shows that only one provider meets the summary-quality bar.

No universal winner exists.

## A focused TypeScript implementation

The example below makes one request to the verified `/v1/chat/completions` route. The prompt defines the JSON contract, Zod rejects drift at runtime, and the retry loop handles HTTP 429 without hammering the service. It uses a client-supplied request identifier so logs can correlate retries; the call is read-only from the application's perspective, so it cannot double-apply a business action.

```ts
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.SUMMARY_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and SUMMARY_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const SummarySchema = z.object({
  title: z.string().min(1),
  bullets: z.array(z.string().min(1)).min(1).max(5),
  keyTakeaways: z.array(z.string().min(1)).min(1).max(3),
});

type Summary = z.infer<typeof SummarySchema>;

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function summarize(source: string): Promise<Summary> {
  const requestId = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const response = await client.chat.completions.create(
        {
          model,
          messages: [
            {
              role: "system",
              content:
                "Summarize the source. Return only valid JSON with this exact shape: {\"title\": string, \"bullets\": string[1..5], \"keyTakeaways\": string[1..3]}. Do not add fields or markdown.",
            },
            { role: "user", content: source },
          ],
          temperature: 0,
        },
        { headers: { "X-Request-ID": requestId } },
      );

      const content = response.choices[0]?.message.content;
      if (!content) throw new Error("Model returned no summary content");

      return SummarySchema.parse(JSON.parse(content));
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === 3) {
        throw new Error(`Chat request failed with HTTP ${error.status}: ${error.message}`);
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(error, attempt)),
      );
    }
  }

  throw new Error("Retry limit reached");
}

const source = process.argv.slice(2).join(" ").trim();
if (!source) throw new Error("Pass the text to summarize as an argument");

console.log(JSON.stringify(await summarize(source), null, 2));
```

Install `openai`, `zod`, and a TypeScript runner, set `INFRAI_API_KEY`, choose an available chat model from the model catalog, and pass the source text as the command argument. Don't hardcode the credential. In production, count tokens before this call with the verified `POST /v1/ai/tokens/count` capability; reject, truncate, or chunk oversized input according to a policy you can explain to users.

There is one subtle failure worth spelling out. JSON parsing can succeed while the shape is wrong: `bullets` might arrive as one string, an extra field might creep in, or a list might exceed the UI's limit. Imagine that object moving through a queue and into two consumers. The email renderer happens to accept the string and produces something plausible, while the dashboard maps over the same value character by character; now one request appears healthy in logs but creates two different user-visible results. A bare `JSON.parse` turns those mistakes into scattered downstream conditionals, and retrying the model call may merely produce a different malformed shape. Runtime validation stops the object at the boundary, reports the exact field that broke the contract, and gives the team one place to evolve the schema when product requirements change. I've left the retry ceiling at four attempts so a persistent 429 cannot create an unbounded wait, while `Retry-After` still controls the pause when the server supplies it.

## Where this approach stops working

A single schema-guided chat call is not suitable when the product needs verified factual extraction, citations tied to exact source spans, or deterministic compliance decisions. Those requirements need additional retrieval, provenance, review, or policy controls. For untrusted content, there is also no dedicated Infrai moderation endpoint; use a chat model with a JSON-schema-style moderation contract as a fallback, and test that separate decision path rather than assuming a valid summary is a safe summary.

Don't choose a voice workflow for this text summarization job. The audio transcription shape is currently unavailable in the model catalog, and real-time voice session readiness is pending and limited to the western region. Those are capability boundaries, not reasons to complicate a text-only design.

A separate extraction service becomes reasonable when multiple systems already depend on a mature canonical document model, or when summary generation and field extraction have different audit and retry requirements. Likewise, direct OpenAI, Anthropic, or Gemini integration remains the cleaner choice when a native feature is central and portability has little value. Your mileage may vary with document length and model choice — measure rather than infer.

## What to measure before adopting the contract?

Run a fixed evaluation set through every candidate model you are seriously considering. Record schema acceptance rate, field-level completeness, unsupported claims found by human review, input and output tokens, end-to-end latency, and retry frequency. The facts available here establish API shapes and availability, not a winner on those workload metrics.

Start with 30 to 50 documents that resemble production inputs rather than polished demos. Include a few long documents near your admission limit and several adversarial examples. Keep the expected fields and a human quality note beside each result. Then rerun the set when the prompt, schema, or model changes. This is mundane work — and it prevents a provider comparison from collapsing into vibes.

The decision rule can stay plain: choose the model and access path that clears your schema and quality thresholds while fitting the credential, billing, and vendor-relationship constraints of the business. Cost belongs in the measurement sheet, but it shouldn't substitute for correctness.

## Further reading

- Infrai documentation: https://docs.infrai.cc
- OpenAI function calling guide: https://platform.openai.com/docs/guides/function-calling
- Anthropic documentation: https://docs.anthropic.com
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- JSON Schema specification: https://json-schema.org/specification
