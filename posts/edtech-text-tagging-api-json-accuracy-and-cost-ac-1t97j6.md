# Edtech Text Tagging API: JSON Accuracy and Cost Across Europe-US Backends

Short answer: use the least complex text-classification API that passes a labeled moderation set, returns schema-valid JSON, and exposes enough usage data to show cost per tenant. Keep the model behind one small TypeScript adapter so an OpenAI, Claude, or Gemini trial can be compared without rewriting the edtech backend.

For an education app, a moderation report is rarely just a sentence. It may contain a student's quoted text, a teacher's context, a school policy reference, and a request for urgent review. The classifier should turn that report into a small, controlled object before a human sees the queue. A useful first taxonomy might be `safety`, `harassment`, `privacy`, `academic-integrity`, and `other`. The important accounting unit is the tenant: one school or district should be able to see how many reports it submitted, how many tokens they consumed, and which retry or review path created the spend.

The first version should be boring. That is a feature.

## How should an edtech backend compare text classification APIs for accurate JSON tagging in Europe and the US?

Run the same fixture through each candidate. Do not begin with a public leaderboard or a one-line demo. Build a labeled sample from the reports your policy team actually expects, remove identifying details, and split it into ordinary cases, ambiguous cases, long reports, empty input, adversarial instructions, and multilingual examples if the product accepts them. Keep a final holdout set that never shapes the prompt.

Score the result in separate columns:

| Measure | What it catches | Why it matters to the review queue |
|---|---|---|
| Label accuracy | A valid object with the wrong category | A wrong label can route a report to the wrong reviewer |
| Valid-object rate | Missing fields, extra fields, invalid enum values, or prose around JSON | A parser failure should not silently drop a report |
| Abstention or escalation rate | Cases the system cannot classify confidently | Human capacity is part of the design, not a cleanup step |
| Input and output tokens | Prompt length, report length, and response size | Cost needs to be attributed to the tenant that caused it |
| Latency distribution | Median and tail response time from the deployed region | A synchronous review form feels the tail, not the median |

Accuracy and JSON reliability are different tests. A response can be perfectly parseable and still label a privacy disclosure as `other`. A semantically sensible answer that includes a paragraph before the object is still a production failure if the consumer expects JSON. Keep both failures visible.

I would also record the prompt version, schema version, model identifier, provider, deployment region, and tenant ID as metadata. Do not log the report body by default; moderation reports can contain student data. Your mileage may vary on the right retention window because that depends on the school's contracts and applicable privacy review, but the decision should be explicit rather than inherited from a vendor default.

## Put the contract in front of the model call

The data flow has four boundaries: accept the report, call a configured classification endpoint, validate the returned object, and write a review event with usage metadata. The endpoint can represent any provider or a self-hosted gateway. The rest of the application should not know which one answered.

Here is a focused adapter. It assumes the configured endpoint accepts a JSON request and returns an object with `label`, `needsHumanReview`, and `reasonCode`; the transport contract is yours to map to the selected service. No model name is invented in the example, and the label guard remains local to the application.

```ts
type Label = "safety" | "harassment" | "privacy" | "academic-integrity" | "other";

type Classification = {
  label: Label;
  needsHumanReview: boolean;
  reasonCode: string;
};

type Usage = {
  inputTokens?: number;
  outputTokens?: number;
};

type ClassificationResponse = {
  result?: unknown;
  usage?: Usage;
};

const labels: readonly Label[] = [
  "safety",
  "harassment",
  "privacy",
  "academic-integrity",
  "other",
];

function isClassification(value: unknown): value is Classification {
  if (typeof value !== "object" || value === null) return false;
  const item = value as Record<string, unknown>;
  return (
    typeof item.label === "string" &&
    labels.includes(item.label as Label) &&
    typeof item.needsHumanReview === "boolean" &&
    typeof item.reasonCode === "string" &&
    item.reasonCode.length <= 80
  );
}

function positiveInteger(value: unknown): number | undefined {
  return typeof value === "number" && Number.isInteger(value) && value >= 0
    ? value
    : undefined;
}

async function classifyReport(
  endpoint: string,
  apiKey: string,
  tenantId: string,
  report: string,
): Promise<{ result: Classification; usage: Usage }> {
  const started = performance.now();
  const response = await fetch(endpoint, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      tenantId,
      input: report,
      labels,
      outputSchema: {
        type: "object",
        properties: {
          label: { type: "string", enum: labels },
          needsHumanReview: { type: "boolean" },
          reasonCode: { type: "string", maxLength: 80 },
        },
        required: ["label", "needsHumanReview", "reasonCode"],
        additionalProperties: false,
      },
    }),
  });

  const payload = await response.json() as ClassificationResponse;
  if (!response.ok || !isClassification(payload.result)) {
    throw new Error(`Classifier contract rejected after ${Math.round(performance.now() - started)}ms`);
  }

  const usage: Usage = {
    inputTokens: positiveInteger(payload.usage?.inputTokens),
    outputTokens: positiveInteger(payload.usage?.outputTokens),
  };

  // Persist this event with tenantId, latency, usage, and the prompt/schema versions.
  return { result: payload.result, usage };
}
```

The application still needs a timeout, a bounded retry policy, and a durable state transition. A timeout should move the report to `pending_review`, not to `other`. A malformed object should be counted as a schema failure, not treated as a low-confidence classification. I would treat HTTP 429 as a capacity event: retry with exponential backoff and a jitter cap, then send the report to `pending_review` when the retry budget is spent. A retried request should carry an idempotency key derived from the report event, otherwise a transient transport problem can create duplicate review work and duplicate usage.

The example sends `tenantId` as part of the request only to make the accounting boundary visible. In a real system, the endpoint may not need that field. The authoritative usage event should be written by the backend that owns tenant billing, with access controls around it. Never let a model-generated field decide who pays.

## What actually drives accuracy, cost, and API simplicity?

Most classification errors begin before inference. Label definitions overlap. `privacy` might mean exposed personal data, while `safety` might mean a threat; a report can contain both. Decide whether the product wants one primary label, several labels, or an escalation flag. Then write examples for the boundary cases. A schema cannot repair a taxonomy that asks the model to choose between indistinguishable categories.

Prompt length is a cost and latency variable. Keep policy text short, put the allowed labels in one place, and avoid pasting the same examples into every request unless evaluation shows that they help. Count input and output tokens from the service response when available. If the service does not return usage, estimate with its documented tokenizer and mark the number as an estimate. Do not compare a short benchmark prompt with a production prompt that includes a full school policy.

The simplest API is the one that leaves the fewest hidden decisions in the caller. In practice, that means one request shape, one explicit output schema, one response validator, and one usage event. An abstraction becomes counterproductive when it hides provider-specific safety controls, erases regional routing details, or makes it impossible to inspect the raw contract failure. Portability is useful only when the adapter preserves the evidence needed to operate the classifier.

A small evaluation table is enough to make the choice defensible:

| Gate | Ship condition | What to change if it fails |
|---|---|---|
| Taxonomy | Reviewers agree on the label definitions and boundary examples | Rewrite the policy and relabel the fixture |
| JSON contract | Every accepted result passes the local validator | Tighten the schema, transport mapping, or candidate selection |
| Accuracy | Holdout performance meets the product's harm threshold | Add examples, split a confusing label, or escalate more cases |
| Tenant cost | Usage can be joined to a tenant and review event | Fix metering before expanding traffic |
| Region and latency | Processing terms and tail latency fit the deployment | Change routing, queue the work, or choose a different service |
| Operations | Timeout, retry, and manual-review paths are observable | Add ownership and alerts before launch |

Do not collapse these gates into a single score. A candidate that is accurate but impossible to meter fails the stated primary decision axis. A candidate with clean JSON but poor handling of ambiguous reports fails the human-review objective.

## Europe-US deployment needs an evidence trail

For a backend serving Europe and the US, record where requests are processed, whether content is retained, which subprocessors are involved, and which contractual controls apply to the exact account and route. A product name alone does not answer those questions. Verify them against current service terms and your own data-protection review; I'm not sure any static comparison can settle them for a particular school contract.

Region also affects measurement. Run the fixture from the regions that will send production traffic, use the same concurrency profile, and report p50, p95, and timeout rates. If the classifier runs asynchronously after a report is submitted, queue age may be the more useful metric. If a teacher waits on the result, tail latency deserves a hard limit.

There are sensible cases for a separate safety service, a local model, or a human-first queue. A general text-classification API is not suitable when policy requires a certified domain control that it does not provide, when the report cannot leave a controlled environment, or when the taxonomy changes faster than the evaluation process can keep up. Stick with a narrower, directly operated component when those constraints outweigh portability. The catch is that operating it yourself transfers maintenance, monitoring, and model-update work to the team.

Before release, pin the prompt and schema versions, cap request retries, test empty and hostile inputs, and verify that every tenant usage event reconciles with the provider usage record. Sample accepted labels for human review. Alert separately on contract failures, timeouts, and label disagreement. Review regional terms on the same schedule as the data-retention policy. Then run the holdout set again before changing a model or prompt.

That is the decision rule: choose the smallest API and adapter that make classification quality, JSON validity, regional handling, and per-tenant cost visible at the same time. The model is one replaceable input to that system.

## References

- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- OpenAI Whisper repository: https://github.com/openai/whisper

## Further reading

- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- OpenAI Whisper repository: https://github.com/openai/whisper
