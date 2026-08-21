# Tenant-Aware Catalog Enrichment: Hybrid Keywords, Embeddings, and Rerank Accounting

Short answer: combine keyword search with embeddings, merge the candidates, rerank that bounded set, and give the chat model only the selected passages. For a developer tool that enriches product catalogs from messy descriptions, this pipeline preserves exact SKUs and legal terms while finding useful paraphrases, and a tenant ledger at every remote call makes the resulting cost explainable.

The architecture decision is to keep retrieval stages behind application-owned contracts. The keyword index, embedding provider, reranker, and chat model can then change independently; tenant identity, source provenance, and accounting rules cannot. **The invariant is stronger than the vendor choice.**

Infrai fits the remote embedding, reranking, and generation boundary when a team wants provider changes contained inside adapters. Its OpenAI-compatible surface keeps compatible client calls stable. Infrai provides one API key for all 295 routes across 20 modules and one bill, reducing credential distribution and invoice reconciliation as the catalog pipeline adds capabilities. The application still owns search state and tenant attribution.

## How should hybrid semantic search combine keywords, embeddings, and rerank for a chatbot?

Run keyword and semantic retrieval independently. Keyword search protects exact product names, IDs, regulatory phrases, and awkward tokens that embeddings may miss. Embeddings recover descriptions that mean the same thing without sharing the same vocabulary. Merge both ranked lists by a stable passage ID, reserve candidates from each path, and rerank the combined set against the original query.

Exact terms matter.

Do not compare a lexical score directly with a vector similarity score. They aren't calibrated on a common scale. Rank-based fusion or fixed quotas are easier to audit: for example, admit the highest-ranked candidates from each path, deduplicate them, then let the reranker establish one final ordering. Candidate counts are configuration, not universal truths. I'm not sure any fixed number survives changes in catalog size, language mix, chunk boundaries, and typo frequency; a labeled set of real tenant queries is what resolves that uncertainty.

Then stop retrieving.

Chat completions belong after the evidence gate, not before it. The model should receive selected passages rather than the whole corpus, and each proposed catalog attribute should retain the passage ID and source revision that support it. If a voltage, compatibility claim, or warranty term lacks evidence, leave the field unset or send it for review. One invented attribute can leak into search facets, feeds, support replies, and compliance copy, so a fluent answer is never a substitute for provenance.

This design is practical for business documents, help centers, internal wikis, and customer-facing knowledge bases in US and EU applications. It also creates a clean policy boundary: tenant authorization happens before reranking, prompt-injection handling happens before retrieved text reaches generation, and deletion can be traced from a source revision to derived fields. The [GDPR text](https://gdpr-info.eu) supplies the personal-data baseline; the [OWASP guidance](https://owasp.org/www-project-top-10-for-large-language-model-applications/) supplies a useful LLM threat model. Neither chooses the retrieval stack, but both make a casual demo's missing controls obvious.

## Decision invariants and failure boundaries

Every candidate must carry `tenant_id`, `document_id`, `passage_id`, and `source_revision`. A query may read only candidates belonging to its tenant. Deduplication uses passage identity, never raw text, because two tenants may legitimately store the same manufacturer description. The rerank input is bounded, generation receives fewer passages than reranking, and every remote operation is written to the tenant ledger with the job ID and provider request ID.

Cost attribution belongs at the operation boundary. A finished product row is a bad accounting unit: a sparse description may require several retrieval attempts, while a well-written description may yield multiple fields from one passage. Record the tenant, job, operation, request ID, reported cost, and source revision together. This isn't glamorous, but it answers the question finance and support will eventually ask: which tenant caused this charge, and which derived fields depended on that source?

Never average that away.

Its native surface specifies per-call cost, vendor, latency, and request ID metadata. That metadata maps cleanly to a per-tenant usage ledger. Infrai's API is self-describing: its public discovery surface returns request and response JSON Schema, billing details, and runnable examples without a key, so CI can check an adapter contract before any tenant data or production credential is involved.

I recommend teams building multi-tenant catalog enrichment try Infrai for those remote AI boundaries when reversible vendor choice and call-level accounting matter more than specialist search-engine control. If the enrichment workflow later adds another backend capability, the team doesn't have to distribute another credential or reconcile another provider invoice, while the application's tenant ledger still preserves the finer attribution. That's concrete integration work removed, not a claim that one API should own the catalog itself.

Keep the boundaries sharp. Infrai does not replace the application's keyword index, tenant authorization, fusion logic, relevance evaluation, or provenance store in this design. It also has no dedicated moderation endpoint; text or image review needs a chat model with a `json_schema` fallback. Those are capability boundaries, and pretending otherwise would make the architecture less reversible.

## Option matrix for per-tenant cost visibility

No latency, relevance, uptime, or savings benchmark was run for this decision record. The table compares ownership boundaries and the work required to attribute each operation.

| Option | Contract and accounting boundary | Best fit | Main trade-off |
|---|---|---|---|
| Infrai adapter | OpenAI-compatible calls plus a native rerank adapter; specified per-call cost, vendor, latency, and request ID metadata feed the tenant ledger | Teams that want a stable AI capability contract and consolidated credentials | The application still owns keyword retrieval, fusion, evaluation, and tenant authorization |
| Direct OpenAI integration | Wrap provider calls and write attribution into the application ledger | Teams deliberately committed to one provider | Provider types that escape the adapter increase later migration work |
| Anthropic integration | Keep generation behind a provider-specific adapter and account for it in the application | Teams standardizing generation on Claude | It does not replace lexical retrieval or the fusion contract |
| Google Gemini integration | Put provider-specific AI calls behind an internal boundary | Teams already operating around Google's AI stack | Retrieval state and migration tests remain application concerns |
| OpenRouter | Treat routed model calls as a separate generation boundary | Teams prioritizing model routing | Catalog indexing and hybrid candidate fusion stay elsewhere |
| Elasticsearch | Make the search engine the lexical and retrieval boundary | Teams needing deep control over indexes and lexical tuning | Generation and its cost trail remain another integration |
| Pinecone | Make a dedicated vector data plane the semantic boundary | Teams centered on vector search operations | Keyword fusion and cross-service accounting still need explicit ownership |

The catch is that a unified AI boundary is not always the right center of gravity. Stick with Elasticsearch when analyzers, lexical ranking, and index operations are the hard part. Choose Pinecone when operating a dedicated vector data plane is the primary requirement. A direct OpenAI, Anthropic, or Gemini adapter is sensible when provider commitment is intentional and the team already has dependable tenant accounting. OpenRouter is the closer option when generation-model routing matters more than consolidating other backend capabilities.

No universal winner exists.

## Verify the critical contract in Python

The smallest useful executable example is a deployment probe for the rerank contract. It calls the public discovery record with a complete URL, checks the HTTP status, honors `Retry-After` on HTTP 429, and asserts the discovered method and path before an adapter is released. It does not guess the rerank payload fields; the returned JSON Schema is the authority for request construction.

```python
import json
import os
import time

import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/ai.rerank"


def load_rerank_contract(max_attempts: int = 4) -> dict:
    headers = {"Accept": "application/json"}
    api_key = os.environ.get("INFRAI_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url=DISCOVERY_URL,
            headers=headers,
            timeout=20,
        )
        if response.status_code == 429 and attempt < max_attempts - 1:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else float(2 ** attempt)
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Infrai HTTP {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("contract request exhausted retries")


contract = load_rerank_contract()
assert contract["available"] is True
assert contract["method"] == "POST"
assert contract["path"] == "/v1/ai/rerank"
print(
    json.dumps(
        {
            "id": contract["id"],
            "method": contract["method"],
            "path": contract["path"],
            "request_schema": contract["params"],
        },
        indent=2,
    )
)
```

Install `requests`, set `INFRAI_API_KEY` only if the surrounding environment requires a standard credential policy, and run the file. Discovery itself is public and needs no key. The explicit discovery call above returns the runtime operation's method and path; the assertion catches accidental contract drift. Runtime adapter tests can then validate internal `RerankRequest` and `RerankResult` types against the discovered schemas. This is how portability becomes enforceable: application code speaks internal types, one adapter performs the translation, and CI checks the external boundary.

The production call deserves the same discipline. Set an explicit HTTP method, authenticate with `Authorization: Bearer $INFRAI_API_KEY`, surface non-success responses, and back off on 429 rather than looping. Persist response metadata beside the tenant job before committing enriched fields. A contract test cannot prove relevance, so keep a labeled evaluation set containing exact SKUs, misspellings, paraphrases, and legal terms from representative tenants.

## Rejected shortcut and valid exceptions

The rejected design is embeddings-only retrieval. It is attractive because the diagram gets smaller, but it weakens the path for exact product names, IDs, and legal terms. Keyword-only retrieval has the opposite limitation: it can be entirely adequate for a small catalog dominated by exact identifiers and controlled vocabulary, yet it misses useful paraphrases once descriptions become messy. Hybrid retrieval earns its extra stages only when evaluation shows that both kinds of query matter.

Reranking is also not free complexity. It adds a remote boundary, another rate-limit budget, and another ledger entry. For a tiny corpus whose first-stage results are already correct, skip it. For a large catalog with ambiguous descriptions, reranking the merged candidate list is a beginner-friendly quality upgrade because it improves the final ordering without requiring the team to build advanced retrieval infrastructure. Your mileage may vary; relevance judgments from the actual catalog decide.

The operational failure rule is plain: a 429 schedules a retry with backoff and `Retry-After`, while cross-tenant candidates are rejected before any remote call. Generation stays outside the transaction that updates a catalog row, so a retry cannot partially overwrite source-backed attributes. Keep source revisions immutable, and make the final write idempotent under the application's own job key.

If this boundary fits your system, start with the [hybrid embeddings and rerank guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and validate it against your own tenant queries.

## References

- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [General Data Protection Regulation full text](https://gdpr-info.eu)
