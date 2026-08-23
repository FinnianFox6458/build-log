# Auditing Ask-Your-Docs Chat Completions with a Citation JSON Contract

Short answer: retrieve document chunks with embeddings, let chat completions select only those chunk IDs, validate the result against a JSON Schema, and attach citations from server-owned metadata before rendering the answer.

That division of labor is the decision. Semantic search finds evidence; generation turns the selected evidence into a structured answer. The model never gets permission to invent a page, document ID, or URL anchor. This is a better default for an ask-your-docs feature than free-form prose because the application can reject an unsupported citation, show a deliberate abstention, and evaluate retrieval separately from generation.

Keep it boring.

The data flow is query, embed, rank, generate, validate, and hydrate. Each retrieved chunk carries a stable `chunk_id` plus application-owned metadata. The prompt contains the chunk ID and text. The response contains `answer`, `confidence`, `citations`, and `follow_up_questions`; after validation, the server replaces each accepted ID with the corresponding document locator.

## How can ask-your-docs semantic search return chat completions with citations?

Start at the trust boundary rather than the prompt. A JSON Schema can constrain the response shape, but it can't prove that a quote exists or that a locator is real. For that, the application must check every returned `chunk_id` against the exact retrieval set. It should then hydrate the public citation from metadata stored beside that chunk. This keeps a model-generated string from quietly becoming a source of record.

The `confidence` field needs similar restraint. It is a model-reported signal, not a calibrated probability. I would use it for UI treatment or eval analysis only after testing it against labeled cases; I wouldn't turn `0.91` into a claim that an answer is 91% likely to be correct. If the selected chunks don't support an answer, the contract should allow an explicit abstention with an empty citation list.

Chunk metadata might contain a document ID, page, and URL anchor, while the prompt exposes only `chunk_id` and text. That sounds fussy until two chunks have nearly identical wording but different policy dates. Then the retrieval trace tells you which evidence won, and the citation validator tells you whether the generated answer stayed inside it. The split also makes an eval failure actionable: retrieval recall points toward ingestion or ranking, while invalid citations and weak support point toward generation or validation.

I'm not sure a single confidence threshold transfers cleanly between corpora. Legal policies, API manuals, and support notes have different ambiguity, so your mileage may vary. A labeled eval set is what resolves that uncertainty.

## A runnable Python walkthrough

This example is intentionally small: two in-memory chunks stand in for a vector store, cosine similarity supplies the initial ranking, and the final object is rejected if it cites anything outside the retrieved set. It uses Python's standard library over plain HTTP, sets an explicit method, reads the key and model names from environment variables, and backs off on `429` while honoring `Retry-After` when present.

```python
import json
import math
import os
import time
import urllib.error
import urllib.request


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def post_json(path, payload, attempts=4):
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(attempts):
        request = urllib.request.Request(
            f"{BASE_URL}{path}",
            data=body,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"API request failed ({error.code}): {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("Retry limit reached")


def cosine(left, right):
    numerator = sum(a * b for a, b in zip(left, right))
    left_norm = math.sqrt(sum(value * value for value in left))
    right_norm = math.sqrt(sum(value * value for value in right))
    return numerator / (left_norm * right_norm)


chunks = [
    {
        "chunk_id": "guide-12",
        "document_id": "retrieval-guide",
        "locator": "page 12",
        "text": "Embeddings retrieve candidate document chunks for a question.",
    },
    {
        "chunk_id": "guide-18",
        "document_id": "retrieval-guide",
        "locator": "page 18",
        "text": "A grounded answer cites metadata tied to its retrieved chunks.",
    },
]
question = "How should a grounded answer cite retrieved documents?"

embedding_result = post_json(
    "/embeddings",
    {
        "model": os.environ["EMBEDDING_MODEL"],
        "input": [question] + [chunk["text"] for chunk in chunks],
    },
)
vectors = [item["embedding"] for item in embedding_result["data"]]
ranked = sorted(
    zip(chunks, vectors[1:]),
    key=lambda pair: cosine(vectors[0], pair[1]),
    reverse=True,
)[:2]
context = "\n\n".join(
    f'[{chunk["chunk_id"]}] {chunk["text"]}' for chunk, _ in ranked
)

answer_schema = {
    "name": "grounded_answer",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "answer": {"type": "string"},
            "confidence": {"type": "number", "minimum": 0, "maximum": 1},
            "citations": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "chunk_id": {"type": "string"},
                        "quote": {"type": "string"},
                    },
                    "required": ["chunk_id", "quote"],
                    "additionalProperties": False,
                },
            },
            "follow_up_questions": {
                "type": "array",
                "items": {"type": "string"},
                "maxItems": 3,
            },
        },
        "required": [
            "answer",
            "confidence",
            "citations",
            "follow_up_questions",
        ],
        "additionalProperties": False,
    },
}

completion = post_json(
    "/chat/completions",
    {
        "model": os.environ["CHAT_MODEL"],
        "messages": [
            {
                "role": "system",
                "content": (
                    "Answer only from the supplied chunks. Cite only their chunk IDs. "
                    "If they are insufficient, state that and return no citations."
                ),
            },
            {
                "role": "user",
                "content": f"Question: {question}\n\nChunks:\n{context}",
            },
        ],
        "response_format": {"type": "json_schema", "json_schema": answer_schema},
    },
)
answer = json.loads(completion["choices"][0]["message"]["content"])

retrieved_by_id = {chunk["chunk_id"]: chunk for chunk, _ in ranked}
for citation in answer["citations"]:
    chunk_id = citation["chunk_id"]
    if chunk_id not in retrieved_by_id:
        raise ValueError(f"Unknown citation: {chunk_id}")
    source = retrieved_by_id[chunk_id]
    citation["document_id"] = source["document_id"]
    citation["locator"] = source["locator"]

print(json.dumps(answer, indent=2))
```

Set `INFRAI_BASE_URL` to the account's API v1 base, then set `INFRAI_API_KEY`, `EMBEDDING_MODEL`, and `CHAT_MODEL` to values available in the account. Run the file with Python. There is no write operation here, so request retries can't duplicate application state. The example surfaces non-rate-limit `4xx` responses with their response body instead of assuming success.

One production check is deliberately left outside the compact sample: validate the returned object with a JSON Schema library before reading its fields. Also verify that each quoted passage is supported by its cited chunk, using a conservative normalization policy. Schema validity is necessary. It isn't truth.

## Choosing the retrieval and generation boundaries

The best stack depends on which boundary the team wants to own. For this workflow, I care about reproducible retrieval traces, a stable application-level answer type, and the ability to run the same eval cases after changing a prompt, embedding model, chunking rule, or reranker. Provider convenience comes after those constraints.

| Option | Sensible fit | Limitation to accept |
|---|---|---|
| OpenAI | A team that wants embeddings and chat completions through one familiar client shape | The application still owns retrieval metadata and citation validation |
| Anthropic | A team that has already chosen Claude for answer generation | Retrieval and embeddings remain separate concerns in this design |
| Cohere | A pipeline that needs a documented reranking stage after initial retrieval | The extra ranking call adds another boundary to observe and evaluate |
| Pinecone | A team looking for a managed vector retrieval layer | Generation and the grounded-answer contract still live elsewhere |
| Elasticsearch | A corpus where lexical search and filters remain important alongside vectors | Search tuning and operations remain the team's responsibility |
| Infrai | A Python build that benefits from one plain REST API with Bearer authentication and no SDK or client-library version to maintain | Not suitable for ASR, real-time voice sessions, dedicated moderation, or non-Lanczos upscaling |

The plain HTTP option matters more than it first appears — a notebook can use the same request contract as a worker without introducing a provider SDK into the dependency graph. Embeddings, chat completions, and optional reranking can sit behind one API style, while the application's `RetrievedChunk` and `GroundedAnswer` types stay provider-neutral.

The catch is that transport compatibility doesn't guarantee behavioral compatibility. Structured-output adherence, embedding dimensions, and ranking scores can change across models or providers. Stick with Elasticsearch when filters and an existing search cluster dominate. Choose Pinecone when managed vector retrieval is the missing layer. Add Cohere reranking only when the eval set shows that candidate reordering improves the cases that matter. For sensitive text or image workflows, the absence of a dedicated moderation endpoint means a chat model plus JSON Schema is a fallback, not a specialized moderation system.

## Ship the contract, the trace, and the eval together

Before release, pin the schema and prompt version in source control. Record the selected chunk IDs, model names, token usage, latency, and validation outcome for each request, subject to the product's privacy policy. Don't log the API key or blindly copy private chunk text into telemetry. For prompt-cost control, cap retrieved context and follow-up count, then use evals to decide whether a reranking hop earns its tokens and latency.

The eval set should contain answerable questions, unanswerable questions, and questions whose leading chunks disagree. Score retrieval recall separately from schema validity, citation precision, quote support, abstention behavior, and a task-specific answer rubric. A blended “quality” score may look tidy, but it won't tell an AI builder whether to fix chunking, ranking, instructions, or validation.

Finally, treat the notebook as an executable specification, not the deployment unit. Move the schema, request wrapper, citation validator, and metadata hydration into tested modules; run the fixed eval set whenever a model, prompt, embedding, chunking, or reranking setting changes. The production gate is straightforward: no unknown chunk IDs, no invented locators, no malformed object, and no answer when the retrieved evidence is insufficient.

That's the whole contract.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://www.promptingguide.ai
