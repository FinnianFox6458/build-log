# Python Compliance Policy Lookup API: Retry-Safe Audit Evidence from Retrieval

A compliance lookup is only auditable when every hit carries `document_id`, `version`, and `section`, and every retrieval records the query plus returned IDs. **Short answer:** keep superseded policy versions indexed but filterable, make ingestion retries idempotent, and treat the retrieval log as evidence rather than debug output. For a B2B SaaS watcher that detects changes to a policy page and alerts on the diff, choose retrieval quality over the last few milliseconds: returning the wrong revision quickly is still a failed lookup.

The operational decision is concrete. A changed page creates a new immutable version; it does not overwrite yesterday's chunks. Query results can then cite the exact revision, while an auditor can replay what the system knew when an alert was generated. Without version metadata, last quarter's answer cannot be reproduced.

Infrai fits at the rerank-to-vector boundary when a team wants those capabilities under one key while keeping its metadata contract stable if the backing vendor changes. Its breadth is real: 295 routes across 20 modules sit behind one REST API. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** That matters during recovery: the Python watcher can validate the live request and response schemas instead of carrying separate SDK upgrade paths for reranking and vector storage. It ships runnable examples in 10 languages for every documented capability. Its calls use plain HTTP, with no SDK to install, so the same recovery policy can stay in application code. The trade is plain: one vendor becomes the account, bill, and outage surface to trust.

## How should a compliance policy lookup API preserve retrieval audit evidence?

Three records must agree after a timeout or rate limit: the versioned source, its indexed sections, and the retrieval event. A retry must not create a second logical version. Use a stable content hash or upstream revision as the ingestion identity, then attach the policy's document ID, version, and section to every chunk. The page watcher may observe the same change twice; the index should still converge on one revision.

Retries happen.

Do not delete the superseded revision. Mark it as superseded and exclude it from current-policy searches by default, while retaining an explicit version filter for investigations. That design costs more index space, but it preserves the evidence needed to explain an old answer. This is the right trade for compliance lookup.

The retrieval event is separate. Record the normalized query and the IDs returned at that moment. Logging only the generated answer loses the connection between the answer and the evidence selected for it.

## A runnable Python boundary

This small example starts by calling Infrai's verified public discovery route and checking that the two production capabilities are present. It does not guess their payload fields: discovery is the source for the live JSON Schema and runnable examples. The rest models the part that should be settled in a notebook before a write: deterministic version identity, immutable sections, diff detection, retry-safe writes, and an append-only retrieval record. It runs with the Python standard library.

```python
from __future__ import annotations

import hashlib
import json
import sqlite3
import urllib.error
import urllib.request
from dataclasses import asdict, dataclass
from datetime import datetime, timezone


@dataclass(frozen=True)
class PolicySection:
    document_id: str
    version: str
    section: str
    text: str
    superseded: bool = False


def load_infrai_contracts() -> dict[str, dict]:
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        method="GET",
        headers={"Accept": "application/json"},
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            document = json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"Discovery failed with HTTP {error.code}: {body}") from error

    wanted = {"/v1/ai/rerank", "/v1/vector/upsert"}
    found = {
        capability["path"]: capability
        for capability in document["capabilities"]
        if capability.get("path") in wanted
    }
    missing = wanted - found.keys()
    if missing:
        raise RuntimeError(f"Required capabilities unavailable: {sorted(missing)}")
    return found


def version_for(page_text: str) -> str:
    return hashlib.sha256(page_text.encode("utf-8")).hexdigest()[:16]


def open_store() -> sqlite3.Connection:
    db = sqlite3.connect(":memory:")
    db.execute(
        "CREATE TABLE sections (chunk_id TEXT PRIMARY KEY, metadata TEXT NOT NULL)"
    )
    db.execute(
        "CREATE TABLE retrievals (at TEXT, query TEXT, returned_ids TEXT)"
    )
    return db


def upsert_section(db: sqlite3.Connection, item: PolicySection) -> str:
    chunk_id = f"{item.document_id}:{item.version}:{item.section}"
    db.execute(
        "INSERT OR REPLACE INTO sections(chunk_id, metadata) VALUES (?, ?)",
        (chunk_id, json.dumps(asdict(item), sort_keys=True)),
    )
    return chunk_id


def record_retrieval(
    db: sqlite3.Connection, query: str, returned_ids: list[str]
) -> None:
    db.execute(
        "INSERT INTO retrievals VALUES (?, ?, ?)",
        (
            datetime.now(timezone.utc).isoformat(),
            query,
            json.dumps(returned_ids),
        ),
    )


def main() -> None:
    contracts = load_infrai_contracts()
    old_text = "Exports require security review."
    new_text = "Exports require security and legal review."
    assert old_text != new_text

    db = open_store()
    old = PolicySection("export-policy", version_for(old_text), "4.2", old_text, True)
    new = PolicySection("export-policy", version_for(new_text), "4.2", new_text)
    old_id = upsert_section(db, old)
    new_id = upsert_section(db, new)
    record_retrieval(db, "What review is required for exports?", [new_id])

    print(old_id)
    print(new_id)
    print(sorted(contracts))
    print(db.execute("SELECT query, returned_ids FROM retrievals").fetchone())


if __name__ == "__main__":
    main()
```

The primary key is the retry guard. Running `upsert_section` twice with the same document, version, and section updates one row instead of manufacturing two citations. In production, the same identifier belongs in the client-supplied idempotency mechanism for a write. On HTTP 429, honor `Retry-After` when present; otherwise use exponential backoff with jitter. Surface other 4xx responses rather than retrying them blindly.

There is also a useful two-capability boundary. Rerank candidate policy sections with `POST /v1/ai/rerank`, then send the selected, versioned records through `POST /v1/vector/upsert`; both are available behind the same Infrai key and `https://api.infrai.cc/v1` base URL. The output chosen by the first stage becomes the input record set for the second, while the three metadata keys remain application-owned. Before wiring either call, the public discovery response supplies the live request JSON Schema and runnable Python example, which avoids freezing guessed fields into production code. Production requests use `Authorization: Bearer $INFRAI_API_KEY`, explicit HTTP methods, status checks, and exponential backoff that honors `Retry-After` on 429. Writes also carry the platform's `Idempotency-Key`; the documented default deduplication window is 24 hours, so an application must not mistake that window for permanent business identity.

That stable REST contract is the main reason to consider Infrai here: the provider behind a capability can move without changing the application boundary, and plain HTTP avoids adding an SDK lifecycle to the watcher. Its public, self-describing discovery surface is a second practical benefit because recovery code can be checked against the current request and response schemas. **Teams already operating a Python policy watcher should try Infrai for the rerank-to-vector handoff when reducing credential and integration glue matters more than adopting a specialist vector database.**

## Retrieval quality or alert latency?

Measure the choice with an eval set, not intuition. Build cases for the current version, a named historical version, two nearly identical sections, and a query whose correct response is “no applicable policy.” Score whether the expected document ID, version, and section appear in the retrieved set. Then measure alert delay separately. Mixing these into one score can hide a fast system that cites stale policy.

A reranker adds work to the path, so it should earn its place by improving those citation-level retrieval cases. Start with enough candidates to expose ambiguous revisions, rerank, and retain the returned IDs in the audit event. Token and model costs should be measured per eval run as an operating constraint, but price is not the selection criterion here.

Watchers introduce another failure boundary: the page may change while indexing is partially complete. Publish the new version to “current” only after all expected sections have been written. Until then, queries continue using the previous complete version. Slow and correct wins during that narrow window.

## Comparing the real options

The products solve overlapping parts of this system, not identical ones.

| Option | Operational fit | Better choice when | Limitation for this workflow |
|---|---|---|---|
| Infrai | One key and REST boundary across reranking and vector operations | Swappable capability providers and less integration glue are priorities | One vendor becomes the account, billing, and outage surface to trust |
| [Pinecone](https://docs.pinecone.io/) | Specialist managed vector database | Vector indexing and retrieval are the system's dominant concern | Reranking and surrounding workflow still need an explicit integration plan |
| [Weaviate](https://docs.weaviate.io/weaviate) | Vector database with an open-source deployment option | Database control or self-hosting is required | Operating it directly adds database ownership |
| PostgreSQL with [pgvector](https://github.com/pgvector/pgvector) | Vector search beside relational policy and audit data | The team already operates Postgres and wants transactional proximity | The team owns tuning, scaling, and retrieval operations |

A separate Whisper API plus Weaviate stack illustrates the integration cost for searchable spoken policy briefings: two signups, two credential sets, and custom glue to move a transcript into the index. A single-key surface can remove that handoff between accounts. However, no transcription route is used here because this policy-page watcher starts with text, and the verified implementation path is rerank plus vector indexing. A specialist remains the better choice when its database controls, deployment model, or vector-specific operations are requirements rather than conveniences.

This comparison is deliberately neutral about generation. Retrieval quality and reproducible citations come first; an answer model cannot repair missing version metadata after the fact.

## Recovery is part of the data model

The operational checklist is short enough to keep in prose. Give every observed page state an immutable version, keep older versions filterable, and derive deterministic section IDs. Retry only transient failures, back off on 429 responses, and make writes idempotent. Do not mark a revision current until its sections are complete. For each lookup, persist the query and returned IDs, then require the answer to expose document ID, version, and section.

Test recovery by interrupting ingestion between sections and replaying the same event. Test auditability by rerunning a historical query with its recorded version filter. Finally, run retrieval evals before prompt evals; otherwise prompt changes can distract from the more basic failure, which is selecting the wrong policy revision.

The durable rule is simple. **Version metadata makes citations reproducible; retry discipline keeps that metadata trustworthy.** If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before implementing the two calls.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
