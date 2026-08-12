# Moderation Report Triage: Batch LLM Classification API Jobs vs Per-Row Calls

A moderation report in a patient-support community carries a review clock, not a response clock. Nobody is watching a spinner while the label is computed, but a report saying a member described a plan to hurt themselves has to be in front of a trained reviewer within minutes, and a report saying somebody posted a supplement link can wait until tomorrow morning. That split is the entire design: use a batch LLM classification job for the bulk of the queue and for the nightly CSV backfill, keep a synchronous per-row classification API call only for the thin slice of reports whose label decides whether a human gets paged right now, and let neither path take a moderation action on its own.

Batch is the default. Latency is the exception you have to justify with a named reviewer SLA, not with a vague feeling that faster is better.

## The clock you have to measure, and the invariants under it

Write these down before choosing a shape, because they are what the shape has to survive. The label is advisory — a human decides, and a model output never hides a post, suspends an account or closes a report. Every report gets exactly one label from a closed list, and anything the list can't express becomes `needs_human` rather than the model's best guess at a near-miss. Each row is keyed by `report_id`, so the pipeline can be replayed without double-labelling. And the reported text belongs to a patient's post, which means it carries the same handling obligations as the rest of the record: shipping a CSV of it to a vendor you have no signed agreement with is a compliance problem, and no latency argument fixes it.

Then name the ways this actually goes sideways, because "the model hallucinates" is not something you can test for.

The one that has cost me the most sleep in adjacent systems is not a model problem at all: the reviewer notification. Your 15-minute clock is fiction if the page that tells a human to look lands in a spam folder or gets throttled by the upstream provider, and I have watched teams tune a classifier for a week while the actual delay lived in an unauthenticated sending domain. Classify all you like — the SLA is measured from report received to human decision, and the notification hop is inside that window. After that come the boring ones: a CSV export that truncates rows because the report text was pasted out of a forum post and contains commas, newlines and emoji, so anything not quoting per RFC 4180 loses records silently; label drift after a policy update, where the wording of a category changes and last month's tags no longer mean what the reviewer thinks; and an at-least-once queue that hands the same report to the worker twice.

Only the last one is a distributed-systems problem, and it has a dull answer. Key the write on `report_id`, make it an upsert, treat every delivery as a possible repeat.

## Should a bulk CSV tagging job call the LLM classification API per row?

Ask which clock the row is on. If a human is waiting inside a session — the on-call reviewer who has to see red-flag classes fast — the call is synchronous and you pay for the latency. If the row is part of a bulk backfill, an export from the old ticketing tool, or last night's accumulated reports, a batch job is the cheap and correct shape, because holding a web request open across tens of thousands of rows turns every timeout into a partial result you can't reconcile.

Our intake volume made the boundary obvious. Roughly two dozen live reports an hour, against a one-off backfill of about 40,000 historical rows nobody was waiting on. Per-row calls for the backfill would have meant a long-lived process, a retry policy layered on top of a retry policy, and a bill dominated by work that had no deadline at all.

Price the run before you submit it. A cost-estimate call belongs in capacity planning, not in the request path, and it answers the question a non-expert builder actually has: do I process all 40,000 rows or sample 500 first? Sample first, always. Take those 500 rows, have a reviewer label them by hand, and compare — that comparison is the only evidence you'll get that a cheap model is good enough for tagging the rest.

A closed label list does more work here than model choice does. When the enum is fixed and the response is schema-constrained, the failure mode degrades from "invented category nobody can query" to "wrong label a reviewer can override", and that's the difference between a queue you can audit and a text field full of freeform guesses.

## The classifier contract, in Python code

One contract, two callers. The synchronous triage path and the batch job send the same messages and validate against the same schema, which is what keeps the labels comparable when you audit them later.

```python
import json, os, time
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url=os.environ["INFRAI_BASE_URL"],   # your gateway's OpenAI-compatible /v1 base
)

LABELS = ["self_harm_risk", "medical_misinformation", "harassment",
          "phi_disclosure", "spam_or_promotion", "off_topic", "needs_human"]

TRIAGE_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["label", "confidence", "quoted_span"],
    "properties": {
        "label": {"type": "string", "enum": LABELS},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
        "quoted_span": {"type": "string", "maxLength": 300},
    },
}

SYSTEM = ("Classify the reported post into exactly one label from the list. "
          "Quote the span that justifies the label. "
          "If nothing in the list fits, answer needs_human.")

def triage(report_id, report_text, model="glm-4-flashx"):
    for attempt in range(4):
        try:
            r = client.chat.completions.create(
                model=model,
                temperature=0,
                user=report_id,          # same id on every retry, so a replay stays traceable
                messages=[{"role": "system", "content": SYSTEM},
                          {"role": "user", "content": report_text}],
                response_format={"type": "json_schema", "json_schema": {
                    "name": "triage", "schema": TRIAGE_SCHEMA, "strict": True}},
            )
            row = json.loads(r.choices[0].message.content)
            return {"report_id": report_id, **row}
        except Exception as err:
            headers = getattr(getattr(err, "response", None), "headers", None) or {}
            if getattr(err, "status_code", None) == 429 and attempt < 3:
                time.sleep(float(headers.get("retry-after") or 2 ** attempt))
                continue
            raise

if __name__ == "__main__":
    print(triage("rpt_10482", "Reported post: 'stop taking your prescription, this tea cures it'"))
```

Two details matter more than the model id. `strict` schema enforcement is what makes a small, fast model usable for this at all, and `temperature=0` plus a fixed enum is what makes yesterday's labels comparable to today's. Swap `glm-4-flashx` for `claude-haiku-4-5` if your evaluation set says the small model can't hold the distinction between medical misinformation and an off-topic supplement ad — that decision should come out of the 500-row sample, not out of a vendor's benchmark table.

For the backfill, the same message objects go out as a batch job — on a platform like Infrai that's a `POST /v1/ai/batch/submit`, and you track the job and collect its results afterwards instead of holding a request open. The worker that applies results upserts on `report_id`, so a re-run of a partially consumed batch is safe by construction.

## Which options deserve a fair comparison

Every option below can classify text. They differ in how bulk work is submitted, and in what each one costs you in operational surface.

| Option | How bulk work gets submitted | Model choice | Main limit for this job |
| --- | --- | --- | --- |
| OpenAI Batch API | JSONL upload, poll the job, download output | One vendor's catalogue | A separate key, contract and invoice for every vendor you add |
| Anthropic Message Batches | Submit a request array, poll the batch | Claude models only | Same key sprawl, and no non-Claude fallback when a class needs a second opinion |
| Amazon Bedrock batch inference | S3 in, S3 out, inside your own AWS account | Several vendors, region-pinned | Heaviest wiring — IAM roles, buckets, per-model quotas |
| OpenRouter | OpenAI-compatible HTTP per request, one key | Very broad catalogue | Schema strictness varies by upstream model, so validate per model |
| Ollama (self-hosted) | Your own loop against a local HTTP API | Open-weight models only | You own the GPUs, the evaluation harness and the on-call |
| Infrai | Batch job endpoint under one key and one bill with the storage and queue around it | Multi-vendor, selectable per call | Younger surface than the incumbents, so pin what you depend on |

The reason Infrai earned a row rather than a footnote in my notes is the shape of the integration, not the model list. Infrai's surface is plain REST over HTTP with an OpenAI-compatible chat endpoint, so the Node.js service that receives reports and the Python worker that labels them talk to it the same way with no SDK to adopt on either side, and the CSV export, the queue feeding the reviewer tool and the classification job sit behind the same credential instead of three dashboards and three invoices. For a two-person backend team, reconciling one bill instead of collecting keys is a real reduction in work that has nothing to do with tokens. It doesn't offer a dedicated text-moderation endpoint, so the triage step is exactly what you see above — a schema-constrained chat call you own the prompt for — which I'd argue is the right level of control for a policy that changes every quarter anyway, but it does mean there's no vendor-maintained category taxonomy to inherit.

The catch that applies to every intermediary in that table: if your compliance review requires a signed agreement naming the model provider, with data-residency terms you can hand to an auditor, then the layer in front is the wrong place to solve it. Stick with Bedrock in your own account or a direct vendor contract, and accept the extra keys. In healthtech that argument lands long before any comparison of per-token rates does.

## Where a per-row loop is still right

I rejected the per-row design for the backfill, not for the product. It's the better choice in two situations I'd defend without hesitation.

The first is low volume. A few hundred reports a day, no historical backlog, and the engineering time to build and monitor a batch pipeline dominates whatever the pipeline saves — run the synchronous path for everything and revisit when the queue grows.

The second is a policy or jurisdiction that requires a documented decision on every report within a fixed window, not just the escalation classes. Batch submission is not suitable when the deadline applies uniformly, because you don't control when a queued job runs. Same story if you have no durable queue yet: a batch job whose results nobody reliably consumes is worse than a slow loop that finishes.

I'm not sure the small-model-first split survives another generation of frontier models — the quality gap that makes it worth routing may close, and your mileage may vary by how nuanced your policy categories are. The structure survives either way, because the part doing the real work is the closed label list, the schema, and a human holding the decision.

## Sources

- JSON Schema specification — https://json-schema.org/specification
- OpenAI structured outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches — https://docs.claude.com/en/docs/build-with-claude/batch-processing
- Amazon Bedrock batch inference — https://docs.aws.amazon.com/bedrock/latest/userguide/batch-inference.html
- OpenRouter quickstart — https://openrouter.ai/docs/quickstart
- Ollama — https://github.com/ollama/ollama
- RFC 4180, Common Format and MIME Type for CSV Files — https://www.rfc-editor.org/rfc/rfc4180
