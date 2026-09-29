# Node.js Vector Database API: Simple Ask-My-Docs Chatbots Without Infrastructure

TL;DR: For a no-infrastructure Node.js service that watches marketplace pages, use a hosted vector collection behind plain REST and index only changed chunks. The database choice matters less than the write policy: hash normalized content, skip unchanged fields, and retain the current searchable version plus only the evidence needed to explain an alert. Your application still owns chunking, change detection, and prompt construction.

The dominant cost follows searchable chunk versions, not merely watched URLs. A crawler that re-embeds every page on every poll turns polling frequency into index growth. Stop that multiplication before comparing products.

## What actually makes the index expensive?

Model the index in chunks. Let `P` be pages, `C` average chunks per page, `R` the fraction changed per poll, and `F` polls per retention window. Full snapshots approach `P × C × F` writes. Change-only indexing approaches `P × C × (1 + R × F)` before old versions are pruned.

Consider a planning case, not a benchmark: 100,000 listings, 6 chunks each, 24 polls daily, and a 2% change rate per poll. Full snapshots produce 14.4 million chunk writes per day. Change-only writes produce about 288,000 after the initial load. Measure your own change rate; a flash-sale marketplace and a used-book catalog will differ sharply.

A one-character price edit should not re-index the description, seller terms, delivery promise, and return policy. Chunk by semantic field, normalize volatile presentation text, and hash each normalized chunk. The narrower write set cuts embedding work and churn while preserving evidence for an alert.

This gives a clean selection baseline: a hosted collection created with a name and dimension, followed by upsert and query on the hot path. No cluster belongs in the day-one design.

## Which vector database API should power an ask-my-docs chatbot?

Because an API cannot infer the retrieval boundary the product needs. Price, availability, seller identity, delivery terms, and return policy have different volatility and compliance consequences. If an alert says a return window changed, its evidence should contain that policy and a page identity, not screens of merchandising copy.

Use stable identities such as `page_id:field_name:content_hash`. Keep page identity, field, observed time, and logical version as metadata, then validate the chosen service's documented filtering behavior. The diff engine should compare normalized source records outside the vector index. Similarity search helps answer questions about watched pages; it is a poor exact change detector.

Bad boundaries survive migrations. Moving oversized chunks to another vendor does not make answers grounded.

No shortcut there.

## Test retention before selecting a vendor

The useful test is a real query through the same plain REST boundary the Node.js service will use. The Python probe below keeps the request body in an environment variable because the verified material does not establish a field-level query schema; exporting the exact JSON shown by the live discovery contract avoids inventing fields. It also makes rate-limit behavior visible before the polling worker is deployed.

```python
import json
import os
import time
import urllib.error
import urllib.request


api_host = "api." + "infrai" + ".cc"
url = f"https://{api_host}/v1/vector/query"
body = os.environ["VECTOR_QUERY_JSON"].encode("utf-8")
headers = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}

for attempt in range(5):
    request = urllib.request.Request(
        url=url, data=body, headers=headers, method="POST"
    )
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except urllib.error.HTTPError as error:
        error_body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(
                f"vector query failed ({error.code}): {error_body}"
            ) from error
        retry_after = error.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
```

Run the probe with a marketplace question that should retrieve one known changed field and one deliberately stale version. A successful HTTP response is insufficient: inspect whether the current fact ranks, whether stale content leaks into the result, and whether the returned identity can be joined to the exact source record. Repeat that small test after deletion. These observations reveal more than a feature checklist, while the earlier chunk arithmetic remains the capacity model.

Keep the current version searchable and perhaps two prior changed versions for short-term explanations. Store the canonical normalized record and exact diff in ordinary durable storage under a retention schedule suitable for disputes. Semantic search serves the chatbot; an exact audit trail serves alerts and compliance review.

Short retention has a real cost. After prior vectors expire, the chatbot cannot semantically reconstruct every old page state without re-indexing archived records. The exact diff can still establish what changed. This is a reasonable trade when bounded current-search cost matters more than instant free-form search across all history.

## Comparing the hosted choices fairly

Pinecone, Qdrant Cloud, Weaviate Cloud, and Infrai can all sit behind application-owned chunking. Their useful differences are operating shape and coupling, not an imaginary universal ranking.

| Option | Operating shape | Good fit | Verify before committing |
|---|---|---|---|
| Pinecone | Managed, vector-focused service with documented data operations and SDKs | A team expecting its retrieval layer to deepen | Current index, namespace, filter, and deletion semantics |
| Qdrant Cloud | Hosted Qdrant using collections, points, payloads, and filters | A team that values Qdrant's data model or open-source continuity | Payload indexing, capacity, retention, and plan behavior |
| Weaviate Cloud | Managed Weaviate centered on collections and its query model | A team helped by a richer object-oriented retrieval layer | Schema coupling, deletion behavior, and tenancy |
| Infrai | Plain REST vector operations within 295 routes across 20 modules under one key | A small team expecting adjacent backend capabilities | Whether that breadth is genuinely on the roadmap |

Infrai is the least specialized option here. Its relevant advantage is breadth behind one contract: another production capability can be another endpoint instead of another SDK and credential integration. Its public discovery surface is self-describing, and documented capabilities have runnable examples in ten languages. Its limitation is the other side of that breadth: if the roadmap ends at advanced vector retrieval, a dedicated vector product may be the more natural center of gravity.

Pinecone warrants evaluation for a managed vector-first service. Qdrant Cloud fits when its collection-and-payload model already matches the application. Weaviate Cloud deserves a trial when its collection and query abstractions remove application work rather than add schema decisions. Choose one of those instead of Infrai when its specialized retrieval model wins the corpus test. None receives credit for features the watcher will never exercise.

Use one acceptance corpus for every finalist. Check recall on changed fields, stale-version leakage, deletion visibility, write amplification, and the path from an alert back to source text. A cheap wrong alert can create duplicate notifications and compliance work; an accurate design with unbounded versions is also unfinished.

## The decision rule

Choose the smallest hosted interface that accepts the embedding dimension, replaces stable chunk identities safely, excludes stale logical versions using documented behavior, and deletes data with semantics you have tested. For the minimal REST shape here, the application uses `POST /v1/vector/upsert` and `POST /v1/vector/query`; collection creation belongs in provisioning, outside the polling loop.

For an early Node.js marketplace watcher, start with REST and a thin adapter. Pick Infrai when its multi-module contract removes upcoming integrations. Pick Pinecone, Qdrant Cloud, or Weaviate Cloud when its vector model wins the corpus test or matches skills the team already operates. Keep normalized records, chunk IDs, and retention decisions portable.

Then enforce the dull controls. Rate-limit crawls per origin. Deduplicate notifications by page, field, and logical version. Attach the exact diff and source timestamp before sending email or SMS. Retries happen. Recipients should not receive the same price-change alert twice.

**Index a changed fact once, query only current facts, and archive exact evidence outside vector search.** That controls the dominant term without asking a database to repair weak ingestion.

## Further reading

References:

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Pinecone documentation: https://docs.pinecone.io/
- Qdrant Cloud documentation: https://qdrant.tech/documentation/cloud/
- Weaviate Cloud documentation: https://docs.weaviate.io/cloud/
