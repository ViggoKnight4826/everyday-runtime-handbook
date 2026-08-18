# 6 Tenant Ledgers Control Reliable Node.js LLM JSON Extraction Cost

Customer-support triage changes reliable LLM JSON extraction cost because every inference must be attributed to the tenant that created the ticket, including retries and rejected output. Short answer: put a tenant-aware admission ledger in front of the extraction path, use token counting to reserve capacity, send urgent tickets through a realtime lane, and move deferrable work to a batch lane only after measuring its queue delay and retry amplification.

This is an accounting decision before it is a model decision. The cheapest quoted token rate doesn't help when malformed JSON is silently retried, a shared queue hides a noisy tenant, or an urgent account-lockout ticket waits behind a backlog. Reliability means producing a validated triage record within the ticket's service objective while preserving enough usage data to reconcile the bill.

Decision status: accepted for support-ticket classification systems that can separate urgent and deferrable work.

## What makes reliable LLM JSON extraction fail before Node.js dispatch?

The first invariant is attribution: no request enters a model queue without `tenant_id`, `ticket_id`, `request_id`, and a pricing snapshot identifier. The snapshot is configuration, not a hard-coded claim about a provider. It records the input-token and output-token rates used for the estimate, the tokenizer identifier, and any batch multiplier that the team has verified for its chosen service. Later pricing changes must create a new snapshot rather than rewriting historical spend.

The second invariant is bounded work. Admission uses an estimated token ceiling, while settlement uses the actual usage returned by the selected runtime when that data is available. The system reserves budget before dispatch, releases unused reservation after settlement, and charges every attempt to the originating tenant. A retry is still consumption. Hiding it under a global operations account makes the cost dashboard look tidy and the engineering decision wrong.

The third invariant is schema validity. A triage result should be a small contract such as category, priority, language, and a bounded confidence value. Parse the response, reject unknown fields, validate types and enums, and store the validation outcome beside usage. Don't let a string that merely resembles JSON cross the boundary. Also keep the original ticket text out of cost logs; customer-support messages can contain email addresses, order details, and authentication context.

The failure boundaries are deliberately narrow:

- A tenant over its reservation limit is deferred or routed to an explicitly approved policy; another tenant's budget is never borrowed implicitly.
- An invalid response is quarantined with its attempt number and validation reason. Retry policy is capped and owned by the application, not hidden inside an unobserved client default.
- A realtime deadline miss doesn't silently become a batch success. It remains a service-level miss even if useful JSON arrives later.
- A token estimate is advisory. Settlement and reconciliation expose estimator drift instead of treating the estimate as an invoice.

This matters in support operations because urgency and deliverability rhyme: a queue can accept work while the useful outcome arrives too late. An OTP that lands after expiry is technically delivered and operationally failed. Ticket triage needs the same distinction.

## Evidence matrix for lane admission

Compare lanes with the same validated dataset, schema, model configuration, and retry cap. The unit of analysis is not cost per call. Use cost per accepted triage record, then split it by tenant and urgency class. That denominator captures invalid output and retries without pretending all tickets have equal token counts.

| Decision factor | Realtime lane | Batch lane | Evidence to record |
| --- | --- | --- | --- |
| User-visible urgency | Fits tickets that must affect an active support session | Fits backlog enrichment and deferred routing | Queue age and accepted-result latency |
| Budget admission | Reserve before each dispatch | Reserve before enqueue and reconcile each item | Reserved, settled, and released amount by tenant |
| Failure accounting | Charge every attempted request | Charge every attempted batch item | Attempts and actual tokens per accepted record |
| Operational pressure | Concurrency and rate limits shape admission | Backlog size and completion window shape admission | Queue depth, oldest age, and throttle events |
| Model comparison | Hold the prompt and schema fixed | Hold the prompt, schema, and batch composition fixed | Validation pass rate and cost per accepted record |

Run the comparison on slices that resemble production: short acknowledgements, long pasted email threads, multilingual tickets, empty bodies, prompt-like text inside quoted replies, and messages containing several issues. Averages hide the tenant with unusually long histories. Report p50 and p95 input tokens per tenant, but don't claim that either percentile predicts the next invoice; it only describes the evaluated sample.

Token counting before dispatch is useful for admission and queue sizing. A tokenizer must match the target model closely enough for that purpose, and tokenizer choice should be versioned with the estimate. The official `tiktoken` repository documents its BPE tokenization library and supported usage. Still, an offline count and provider-reported usage can differ because the complete serialized request may include schema or other framing. The resolution is mundane: retain both values, calculate drift by model configuration, and adjust the reservation margin from observed records.

I'm not sure a batch lane will reduce cost for a particular support workload until the service's current terms and the workload's accepted-result rate are tested together. A lower configured batch multiplier can be erased by stale tickets, duplicate submissions, or extra repair attempts. Your mileage may vary — especially when tenants have very different message lengths.

## Reference implementation of the ledger boundary

The application can be Node.js while the policy below remains language-neutral; this Python reference makes the accounting transitions explicit without binding the design to a commercial SDK. In production, the ledger writes, reservation check, and idempotency claim belong in one transactional boundary. The model adapter should return raw output plus usage, never mutate tenant balances itself.

```python
from dataclasses import dataclass
from decimal import Decimal, ROUND_UP
from enum import Enum
import json
from typing import Callable


class Priority(str, Enum):
    LOW = "low"
    NORMAL = "normal"
    URGENT = "urgent"


@dataclass(frozen=True)
class PriceSnapshot:
    snapshot_id: str
    input_per_million: Decimal
    output_per_million: Decimal
    lane_multiplier: Decimal


@dataclass(frozen=True)
class Usage:
    input_tokens: int
    output_tokens: int


@dataclass(frozen=True)
class Triage:
    category: str
    priority: Priority
    language: str
    confidence: Decimal


def token_cost(usage: Usage, price: PriceSnapshot) -> Decimal:
    million = Decimal(1_000_000)
    raw = (
        Decimal(usage.input_tokens) * price.input_per_million
        + Decimal(usage.output_tokens) * price.output_per_million
    ) / million
    return (raw * price.lane_multiplier).quantize(
        Decimal("0.000001"), rounding=ROUND_UP
    )


def validate_triage(raw: str) -> Triage:
    value = json.loads(raw)
    expected = {"category", "priority", "language", "confidence"}
    if not isinstance(value, dict) or set(value) != expected:
        raise ValueError("triage output has missing or unknown fields")

    confidence = Decimal(str(value["confidence"]))
    if not Decimal("0") <= confidence <= Decimal("1"):
        raise ValueError("confidence must be between zero and one")
    if not all(isinstance(value[key], str) for key in ("category", "language")):
        raise ValueError("category and language must be strings")

    return Triage(
        category=value["category"],
        priority=Priority(value["priority"]),
        language=value["language"],
        confidence=confidence,
    )


def process_ticket(
    *,
    tenant_id: str,
    ticket_id: str,
    request_id: str,
    reserved_tokens: Usage,
    price: PriceSnapshot,
    reserve: Callable[[str, str, Decimal], None],
    invoke: Callable[[], tuple[str, Usage]],
    settle: Callable[[str, str, str, Decimal, Decimal, Triage], None],
) -> Triage:
    reserved_cost = token_cost(reserved_tokens, price)
    reserve(tenant_id, request_id, reserved_cost)

    raw, actual_usage = invoke()
    triage = validate_triage(raw)
    actual_cost = token_cost(actual_usage, price)
    settle(
        tenant_id,
        ticket_id,
        request_id,
        reserved_cost,
        actual_cost,
        triage,
    )
    return triage
```

There are two traps in this compact example. First, `request_id` is the idempotency key for a single logical attempt; a retry receives a linked attempt identity so its usage stays visible. Second, rounding occurs after input and output charges are combined. The chosen precision is an internal ledger policy, not a claim about how any provider invoices. Keep raw token counts so finance can reproduce calculations under the exact stored snapshot.

The `invoke` boundary also makes model comparison less theatrical. Replay a frozen, appropriately redacted evaluation set through adapters, validate with the same function, and write outcomes into the same ledger shape. A candidate that emits valid records on easy tickets but fails long-thread or multilingual slices hasn't earned the realtime lane. Fast is irrelevant when the record can't be consumed.

Deployment should begin in shadow accounting: make the existing routing decision, run the new estimator without dispatching duplicate model work, and compare reservations with already available usage records. Then canary one tenant with an explicit spend ceiling. Alert on reservation exhaustion, estimator drift, validation failures, retry count, oldest queue age, and missing settlements. The missing-settlement alert is crucial because an accepted reservation without a terminal entry turns a temporary capacity hold into an accounting leak.

Keep it boring.

## Why shared reconciliation was rejected

The rejected option is one shared realtime queue with a monthly global budget and invoice-level reconciliation. It is attractive because it has fewer moving parts. It is not suitable when tenants need separate cost visibility, one tenant can submit a burst of long threads, or support priority must influence latency. By the time an invoice reveals the skew, the application has lost the request-level evidence needed to explain it.

Stick with that simpler design when the system has one internal cost center, low and predictable volume, no contractual tenant reporting, and staff can tolerate coarse reconciliation. It can also be the right first stage for a tightly limited prototype. The catch is that migrating later requires stable identifiers now: preserve tenant, ticket, request, model configuration, lane, and usage fields even if the first dashboard ignores most of them.

A batch-only design is also rejected for active-session triage because waiting for a completion window can violate the operational purpose of the classifier. It remains valid for nightly taxonomy cleanup, historical trend extraction, and reprocessing closed tickets where queue age is an explicit trade-off. Conversely, realtime-only is wasteful operationally when a large backlog has no immediate consumer; use it when every accepted result feeds a person or automation that is waiting now.

The decision rule is plain: choose the lane per ticket deadline, choose a model only after schema-valid evaluation, and judge cost per accepted record per tenant. Revisit the record when ticket length distribution, retry behavior, service terms, or support objectives change.

## References

- https://github.com/openai/tiktoken
- https://openrouter.ai/docs
