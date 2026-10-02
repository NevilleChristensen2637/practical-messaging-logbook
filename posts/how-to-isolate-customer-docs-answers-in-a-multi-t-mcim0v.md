# How to Isolate Customer Docs Answers in a Multi Tenant Node.js SaaS

Short answer: in a multi-tenant Node.js ask-your-docs SaaS, derive customer scope from authenticated server context, attach it to every indexed chunk, and make the retrieval adapter require that scope. Then validate the generated answer as data, including its citations, before returning it. A namespace or metadata filter can narrow a search, but neither protects data when application code is free to omit the customer boundary.

For a one-person SaaS, this is a revenue-per-hour decision. I want one small contract that blocks an entire class of cross-customer mistakes, stays testable, and lets me ship weekly. Retrieval tuning can wait. Customer isolation cannot.

## How should a multi tenant Node.js ask docs SaaS enforce scope?

The boundary is the server-owned `tenantId`, not a tenant value sent in JSON, a document label produced by a model, or an instruction in the prompt. The request handler authenticates once, obtains that identifier from trusted context, and passes a scoped capability down to indexing and search.

This distinction matters when a support user asks, "What does our cancellation policy say?" Semantic similarity may place another customer's policy near the query. Retrieval relevance and authorization answer different questions. The first asks which text is close. The second asks which text may be seen. Authorization must win before any chunk reaches answer generation.

I would use both a physical namespace and an exact metadata predicate when the storage adapter supports them. That is defense in depth, not proof. The decisive control remains an interface that cannot perform unscoped search. If the backing system offers only metadata predicates, the same contract still works; the adapter owns the translation. This keeps storage choice outside the request handler and outsources undifferentiated plumbing to one narrow module.

## Build the smallest scoped retrieval path

The following TypeScript uses no client-supplied customer identifier. The branded `TenantId` makes accidental mixing harder during development, although runtime validation at the authentication boundary is still required.

```ts
type TenantId = string & { readonly __tenantId: unique symbol };

type Chunk = {
  id: string;
  tenantId: TenantId;
  documentId: string;
  text: string;
};

type Hit = Pick<Chunk, "id" | "documentId" | "text"> & { score: number };

interface VectorIndex {
  upsert(namespace: string, chunks: Chunk[]): Promise<void>;
  query(input: {
    namespace: string;
    vector: number[];
    filter: { tenantId: TenantId };
    limit: number;
  }): Promise<Hit[]>;
}

interface Embedder {
  embed(text: string): Promise<number[]>;
}

class TenantKnowledge {
  constructor(
    private readonly tenantId: TenantId,
    private readonly index: VectorIndex,
    private readonly embedder: Embedder,
  ) {}

  async search(question: string): Promise<Hit[]> {
    const vector = await this.embedder.embed(question);
    return this.index.query({
      namespace: `tenant:${this.tenantId}`,
      vector,
      filter: { tenantId: this.tenantId },
      limit: 6,
    });
  }
}
```

The important line is not `limit: 6`. It is the constructor: a `TenantKnowledge` instance is already scoped. Callers never receive a general-purpose index. The safe path is shorter.

Indexing needs the identical rule. The ingestion worker should receive trusted job metadata, overwrite any tenant field found in the document payload, and write the same `tenantId` into namespace and metadata. Stable document and chunk identifiers make a retried ingestion job replace the intended records instead of multiplying them. RFC 9110 defines idempotent methods by the intended effect of repeated identical requests; a queue worker can adopt the same operational goal even though it is not an HTTP method.

**Generated output needs a second gate.**

Tenant filtering protects retrieval. It does not prove that the answer has the shape the support UI expects. Treat generation output as untrusted input and accept only a narrow object with an answer, a disposition, and citations referring to the retrieved set.

```ts
type Answer = {
  answer: string;
  disposition: "answered" | "insufficient_context";
  citations: Array<{ chunkId: string; documentId: string }>;
};

function parseAnswer(raw: unknown, hits: Hit[]): Answer {
  if (typeof raw !== "object" || raw === null) throw new Error("invalid answer");
  const value = raw as Record<string, unknown>;
  if (typeof value.answer !== "string") throw new Error("invalid answer text");
  if (value.disposition !== "answered" && value.disposition !== "insufficient_context") {
    throw new Error("invalid disposition");
  }
  if (!Array.isArray(value.citations)) throw new Error("invalid citations");

  const allowed = new Map(hits.map((hit) => [hit.id, hit.documentId]));
  const citations = value.citations.map((item) => {
    if (typeof item !== "object" || item === null) throw new Error("invalid citation");
    const citation = item as Record<string, unknown>;
    if (typeof citation.chunkId !== "string" || typeof citation.documentId !== "string") {
      throw new Error("invalid citation fields");
    }
    if (allowed.get(citation.chunkId) !== citation.documentId) {
      throw new Error("citation was not retrieved");
    }
    return { chunkId: citation.chunkId, documentId: citation.documentId };
  });

  if (value.disposition === "answered" && citations.length === 0) {
    throw new Error("answered responses require a citation");
  }
  return { answer: value.answer, disposition: value.disposition, citations };
}
```

Fail closed. A malformed object becomes an internal failure or a controlled `insufficient_context` response; it does not get patched with a regex and shown to the user. The validator also prevents a syntactically valid citation from naming a chunk that was never retrieved.

Structured output correctness now serves operations too. The UI branches on two explicit states. Observability can count validation failures without logging private chunk text. Support agents get document links that were actually in context.

## Test omissions before the happy path

Many unit tests prove that Customer A can retrieve Customer A's handbook. The higher-value test proves the adapter cannot be called without a tenant at all. Add a fake index that records every query, then assert that namespace and exact filter agree. Seed two tenants with nearly identical cancellation documents and verify that each scoped instance sees only its own record.

Keep the regression suite compact:

1. Reject a missing authenticated tenant before embedding the question.
2. Ignore a conflicting tenant field in the request body.
3. Verify namespace and metadata scope on every query and upsert.
4. Reject citations outside the retrieved hit set.
5. Return `insufficient_context` when no authorized chunk supports an answer.
6. Retry the same ingestion input and verify its intended final state is unchanged.

Do not put document text, embeddings, or generated answers into routine logs. Record opaque tenant and request correlation identifiers, retrieval counts, validation outcomes, latency, and retry attempts. The useful dashboard is boring: it tells you where a request stopped without becoming a second private knowledge base.

## What changes at scale

Keep the scoped interface and change the machinery behind it. Larger tenants may justify separate indexes or storage accounts for operational isolation, while small tenants may share infrastructure with mandatory filters. That is a capacity and blast-radius choice. It should not alter handler code or weaken the authorization contract.

At higher volume, add property-based tests for scope propagation, admission checks for ingestion jobs, bounded retries, and a dead-letter path keyed by idempotency data. Separate answer-quality evaluation from isolation tests. A mediocre answer is a product problem; a cross-customer answer is a security problem. Combining their metrics hides the priority.

There is a real trade-off in using namespace plus metadata: two fields can drift. Generate both inside one adapter from the same trusted `TenantId`, assert their agreement in tests, and expose one scoped operation to the rest of the application. A solo operator has little time for policy spread across handlers. Centralization buys back that time.

Ship the narrow contract first. Tune retrieval later.

## Further reading

- [RFC 9110 HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Open-source speech recognition implementation](https://github.com/openai/whisper)
