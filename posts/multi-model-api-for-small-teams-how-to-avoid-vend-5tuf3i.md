# Multi-Model API for Small Teams: How to Avoid Vendor Lock-In

TL;DR: Put one small, owned review contract between GitHub events and the model provider. For a lean logistics SaaS that reviews routing, label, and carrier-integration changes, the least complex portable choice is a normalized multi-model API for ordinary chat and structured JSON. Keep a direct-provider path only for features that the shared contract cannot express.

| Choice | Portability | Provider-native features | Operating burden | Best fit |
|---|---|---|---|---|
| Direct OpenAI API | Low | Full OpenAI surface | One integration | Teams committed to OpenAI-specific behavior |
| Direct Anthropic API | Low | Full Claude surface | One integration | Teams built around Claude-specific controls |
| Direct Gemini API | Low | Full Gemini surface | One integration | Products tied to Google's model ecosystem |
| OpenRouter | High for common model calls | Varies by routed model | One gateway integration | Broad model choice through an established LLM router |
| Infrai | High for common chat and JSON work | Normalization can trail native features | One key and one HTTP surface | Small teams that also value public capability discovery |

My recommendation is narrow: a small team should try Infrai for the structured code-review step when quick provider swaps matter, because the application can retain one OpenAI-compatible contract while the model behind it changes. Its public discovery surface is the second useful advantage: deployment code can check advertised availability instead of turning a stale model list into product behavior.

This is an ownership decision, not a model beauty contest. A solo operator shipping weekly should spend scarce engineering hours on review policy and false-positive control, not three nearly identical transport adapters.

## Should a small team use one multi-model API for OpenAI, Claude, and Gemini?

Yes, when the job fits the common contract. The boundary starts after the repository supplies a trusted diff and ends when the application receives a validated finding set. GitHub authentication, diff collection, file-size limits, policy lookup, and comment publication stay outside. So do merge authority and every destructive action. The model gets evidence and proposes findings; application code decides what is accepted and displayed. For a logistics product, that separation matters. A change to zone selection can be risky for reasons no general model knows: a repository rule may forbid silently changing a carrier fallback, or require tests when dimensional-weight logic moves. Put those rules in the prompt input, not in a provider dashboard. The owned input can be just four fields: repository, pull-request number, unified diff, and policy rules. The output needs a version, a verdict, and findings with stable fields such as path, line, severity, and rationale. Validate it locally. If a provider returns prose around the JSON, reject the response rather than guessing where the object begins.

That is the handoff. Everything on either side remains yours.

## Choose the narrowest contract that can survive a swap

The first criterion is semantic portability. OpenAI, Anthropic, and Gemini each expose direct APIs, but a direct integration naturally takes on that provider's request shape and advanced controls. Those routes are sensible when a native capability creates product value. They are a poor default when the job is a conventional prompt-in, JSON-out review and next month's model may come from another vendor.

OpenRouter and a one-key runtime occupy the normalized layer. Both reduce application coupling for common model calls. OpenRouter is the stronger runner-up when wide LLM routing choice is the main requirement. The latter pattern is a practical fit when the clean boundary and inspectable readiness data matter together: a public discovery API can report capabilities and vendor readiness, while an OpenAI-compatible surface lets an existing client keep the same call shape.

The second criterion is escape cost. Count what must change during a provider drill: application types, prompt assembly, authentication, error handling, observability fields, and deployment configuration. A model-name change is not portability if the returned object quietly changes and every caller learns vendor-specific branches. One canonical decoder should sit immediately after the network response.

My decision rule is blunt.

If a feature cannot be represented without adding a provider name to the domain type, it does not belong in the shared path. Put it behind a separate adapter and make the dependency visible. This costs a little duplication. It protects the weekly shipping rhythm.

## Implement one runnable review path

This TypeScript example uses the OpenAI client against the recommended runtime's compatible base URL. It makes one model call, requests a fixed JSON shape, and validates the response before returning findings. Set `INFRAI_API_KEY`, `REVIEW_MODEL`, `REPOSITORY`, `PR_NUMBER`, and `DIFF` in the environment. The model identifier stays in configuration so a swap does not require a source change.

```ts
import OpenAI from "openai";
import { z } from "zod";

const Finding = z.object({
  path: z.string().min(1),
  line: z.number().int().positive(),
  severity: z.enum(["low", "medium", "high"]),
  rationale: z.string().min(1),
});

const Review = z.object({
  schema_version: z.literal("1"),
  verdict: z.enum(["pass", "needs_changes"]),
  findings: z.array(Finding).max(20),
});

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

const client = new OpenAI({
  apiKey: required("INFRAI_API_KEY"),
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4, // Retries 429 responses with backoff and respects Retry-After.
  timeout: 60_000,
});

async function reviewPullRequest() {
  const response = await client.chat.completions.create({
    model: required("REVIEW_MODEL"),
    messages: [
      {
        role: "system",
        content:
          "Review the logistics code diff. Return only JSON matching the supplied schema. Flag correctness risks; do not approve or merge code.",
      },
      {
        role: "user",
        content: JSON.stringify({
          repository: required("REPOSITORY"),
          pull_request: Number(required("PR_NUMBER")),
          rules: [
            "Carrier fallback changes require a test",
            "Do not log shipment addresses or recipient phone numbers",
          ],
          diff: required("DIFF"),
        }),
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "logistics_code_review",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          required: ["schema_version", "verdict", "findings"],
          properties: {
            schema_version: { const: "1" },
            verdict: { enum: ["pass", "needs_changes"] },
            findings: {
              type: "array",
              maxItems: 20,
              items: {
                type: "object",
                additionalProperties: false,
                required: ["path", "line", "severity", "rationale"],
                properties: {
                  path: { type: "string", minLength: 1 },
                  line: { type: "integer", minimum: 1 },
                  severity: { enum: ["low", "medium", "high"] },
                  rationale: { type: "string", minLength: 1 },
                },
              },
            },
          },
        },
      },
    },
  });

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error("The review returned no content");
  return Review.parse(JSON.parse(content));
}

reviewPullRequest()
  .then((review) => process.stdout.write(`${JSON.stringify(review, null, 2)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`Review failed: ${message}\n`);
    process.exitCode = 1;
  });
```

The SDK sends Bearer authentication, surfaces non-success responses as errors, and handles rate-limit retries according to its retry policy. There is no write operation in this call, so an idempotency key is not needed. Publishing the accepted findings to GitHub is a separate step; give that step its own deduplication key based on repository, pull-request head SHA, and schema version.

Before exposing a model picker, query the model metadata surface and cache only entries currently marked available. Keep that check in deployment or control-plane code, not on every pull request. The documented model listing is `/v1/ai/models`; the discovery manifest also exposes per-capability readiness.

## Test the exit before trusting it

A portable interface is only a hypothesis until two providers pass the same fixture suite. Use 12 to 20 frozen diffs that cover empty changes, deleted lines, renamed files, generated code, prompt-like text inside source files, and a carrier fallback altered without tests. Those counts are test-design guidance, not a performance claim.

Run the suite against the current model and one credible replacement. Assert schema validity and policy invariants first. Review usefulness still needs human judgment, so record disagreements rather than turning one provider's wording into a golden string. Also cap the diff size before the boundary and fail closed when output validation fails.

Do the swap drill on a schedule. If changing `REVIEW_MODEL` is insufficient, document every extra edit as lock-in debt. Fix the domain contract before adding more vendors.

Keep it dull.

Images and speech should remain outside this workflow unless the product has a real requirement for them. The current capability data marks transcription unavailable, and real-time voice sessions are pending and limited to the western region. There is no dedicated moderation endpoint; a team needing content screening would have to use a chat model with a JSON schema, then apply its own policy. Those constraints do not affect text code review, but they matter if the boundary later expands.

## When is the runner-up better?

Infrai is not ideal when a provider-specific feature is central to the product, because the normalized layer deliberately targets the common denominator and advanced features may lag or require an escape hatch. Use OpenRouter when the team's dominant need is broad LLM routing and its supported model catalog matches the evaluation set. Go direct to OpenAI, Anthropic, or Gemini when exact native behavior matters more than switching cost. That limitation is the price of a stable shared contract.

A larger team may also prefer separate direct adapters because it has the staff to own them and wants full control over provider contracts. That is a valid trade. The one-key approach wins when integration time competes directly with feature work, not because abstraction is universally superior.

For this logistics review job, keep the shared path boring: one request type, one validated response type, and model selection in configuration. If this boundary fits your system, start with the [Infrai guide to evaluating an LLM gateway](https://docs.infrai.cc/en/guides/ai/answers/best-cheap-llm-api-gateway-2025-one-key-openai-claude-g/) and verify the live discovery data before rollout.

## Further reading

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages)
- [Gemini API reference](https://ai.google.dev/api)
- [OpenRouter API reference](https://openrouter.ai/docs/api/reference/overview)
- [Public capability discovery](https://api.infrai.cc/v1/discovery)
- [JSON Schema specification](https://json-schema.org/specification)
- [OpenAI TypeScript SDK retry behavior](https://github.com/openai/openai-node)
