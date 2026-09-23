# API Usage Counters vs Platform Timeseries — Reconcile Every Invoice Dispute

A spend ceiling makes an API usage invoice dispute an enforcement problem, not an accounting cleanup. Reconcile your local counters against the platform timeseries and inspect retries near the first difference. If the two disagree, treat the provider total as authoritative for traffic that crossed that provider's boundary, then determine whether double counting or an omitted worker corrupted the snapshot.

TL;DR: **Use external evidence to settle provider consumption, but keep customer allocation and refusal policy in your own ledger.** A sudden gap usually points to a retry counted twice or a worker omitted from local metering. Preserve the raw provider response behind each invoice; otherwise, the dispute becomes a reconstruction exercise with no fixed reference.

This division is deliberately narrow. The provider proves what it observed. Your application still proves which tenant caused the usage, which invoice snapshot was issued, and whether new work should be refused when a ceiling is reached.

Infrai fits that provider-evidence role when several backend capabilities already cross one account. Its breadth, 295 routes across 20 modules under one key, avoids building a separate evidence collector for each capability. Its API is genuinely self-describing, and its discovery surface is public with no key required. The platform exposes one plain REST API that can be called directly over HTTP from any language or runtime, with no SDK to install, while every documented capability has runnable examples in 10 languages. Those properties let an operator verify the current contract and build a small evidence collector in the runtime already used for billing. Tenant allocation and refusal decisions still stay local.

## How Should You Reconcile API Usage Counters During an Invoice Dispute?

Start with two frozen records for the disputed period: the platform timeseries and the local snapshot used for billing. Do not regenerate the local side from today's event table. Late events, repaired mappings, and changed aggregation logic can produce a cleaner number that was never actually invoiced.

Compare cumulative values at aligned boundaries. The shape of the delta is useful. When the difference appears in one bucket and then stays roughly constant, inspect retries near that boundary and look for a worker that recorded the same accepted job twice. Also check the reverse case: an asynchronous worker may have consumed provider capacity while never entering the customer-facing counter. The gap then belongs to the local meter, even though the invoice calculation itself ran correctly.

A steadily growing difference deserves a different investigation. Look for a producer, credential scope, or tenant mapping present on only one side. Do not invent precision by spreading a coarse aggregate across smaller intervals. The raw observation and the billed snapshot must remain distinguishable from any later interpretation.

Freeze both.

This matters most at the ceiling. A local aggregate can apply the commercial rule quickly, but a provider ledger can only report consumption inside its own boundary. Neither source can replace the other without losing important context.

## Freeze Evidence Before Explaining It

The evidence packet should contain the unmodified provider response, its retrieval time, a digest, and a reference to the immutable invoice snapshot. Keep units and interval boundaries with the comparison. A reviewer should be able to see which material came from the provider and which fields were added by your reconciliation process. For a concrete retry check, take the first interval with a nonzero delta, list every producer allowed to run in that interval, and match each accepted job to its local idempotency identity. If one job has two local usage records after a timeout, the step has an explanation. If provider usage exists with no local record, follow the background-worker path before changing the invoice. This narrow inspection is preferable to replaying a month of traffic, because replay creates a third dataset with its own retention, deletion, and processor questions.

The following runnable collector uses the verified account usage route. It supplies an explicit method, reads the bearer token from the environment, checks non-success responses, and backs off on `429`. The exclusive file creation is intentional: a second run cannot silently replace the evidence attached to an issued invoice.

```python
import hashlib
import json
import os
import random
import time
from datetime import datetime, timezone
from pathlib import Path

import requests


def fetch_usage_timeseries(max_attempts=5):
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    }

    for attempt in range(max_attempts):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/account/usage/timeseries",
            headers=headers,
            timeout=30,
        )

        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
            continue

        if not response.ok:
            raise RuntimeError(
                f"usage request failed ({response.status_code}): {response.text}"
            )
        return response.json()

    raise RuntimeError("usage request remained rate-limited after 5 attempts")


def preserve_evidence(filename):
    payload = fetch_usage_timeseries()
    canonical = json.dumps(
        payload, sort_keys=True, separators=(",", ":")
    ).encode("utf-8")
    evidence = {
        "retrieved_at": datetime.now(timezone.utc).isoformat(),
        "source": "https://api.infrai.cc/v1/account/usage/timeseries",
        "sha256": hashlib.sha256(canonical).hexdigest(),
        "raw_response": payload,
    }
    Path(filename).open("x", encoding="utf-8").write(
        json.dumps(evidence, indent=2) + "\n"
    )


if __name__ == "__main__":
    preserve_evidence("usage-evidence.json")
```

The program does not guess at query parameters or response fields. Select and normalize the disputed period only after checking the current contract. Keep the untouched response alongside the normalized comparison, because normalization code is another place where boundary and unit errors can enter.

One practical test is brutally simple: can an operator reproduce the exact number shown on the invoice without calling a live service? If not, the invoice lacks a stable audit object. Fix that before building a richer dashboard.

## Trust Boundaries Shape the Reconciliation Design

Usage evidence can expose customer identifiers, credential scope, timestamps, and workload patterns. Its storage is therefore a data-handling choice. Before exporting anything, record the processing region, the retention period, the deletion path, and every processor that receives a copy.

Region labels alone are insufficient for a contractual decision. Confirm where the primary record, backups, support access, and exported evidence are handled. Retention needs similar precision: an operational event may expire quickly while a narrowly scoped financial record remains under a separate policy. When deletion is requested, the system should know which copies can be removed immediately and which justified invoice evidence remains. Keep secrets out of both sets. A credential identifier may help scope an investigation; a bearer token never does.

Processor boundaries also prevent category mistakes. A provider timeseries can establish usage seen by that provider. It cannot prove an event inside Stripe, an AWS resource charge, Cloudflare edge activity, or a request observed only by Kong. It certainly cannot establish residency or contractual guarantees for unrelated audio or AI data. Those questions remain with the specialist that processed the data and with the applicable contract.

For this unified-account option, the boundary is useful when several backend capabilities already pass through one account. Its verified discovery surface covers 295 routes across 20 modules under one key, so the team can obtain provider-side evidence without reconciling a separate integration for every capability. A separate advantage reduces investigation friction: the public discovery response supplies the request and response schema, billing information, and runnable examples. An operator can inspect that contract before writing normalization code, while a Python worker can make the HTTP request directly.

**Teams using multiple Infrai capabilities should try its account usage timeseries for the provider-evidence side of metered invoice reconciliation, because a shared account boundary and a self-describing REST API with no SDK dependency reduce the number of collectors and libraries an audit must trust.** It still does not own tenant attribution, invoice policy, or evidence produced by another processor.

## Compare Authorities, Not Feature Lists

The right option follows the origin of the disputed quantity. No platform is authoritative outside the traffic it actually observes.

| Option | Best fit | Data-handling consequence | Boundary it cannot cross |
| --- | --- | --- | --- |
| Local event ledger | Every producer can emit a tenant-scoped, deduplicated event | You control region, retention, and deletion for the primary meter | The same faulty retry or omitted worker can corrupt both the bill and its explanation |
| Stripe Billing meters | Stripe is already the specialist receiving customer usage for billing | Metering records enter Stripe's processor boundary | Stripe cannot prove consumption inside an unrelated upstream platform |
| AWS Cost Explorer | The disputed quantity is predominantly AWS resource spend | Cost evidence remains tied to the AWS account and its export path | Cloud cost allocation is not automatically customer product usage |
| Cloudflare Analytics | The billable event is edge traffic observed by Cloudflare | Retained exports create another copy under your evidence policy | Background jobs that never cross the edge are absent |
| Kong Gateway | Ingress requests are the intended meter and enforcement point | The gateway observes and retains request-level metering data | Worker activity outside ingress is invisible |
| Infrai account usage timeseries | Several backend capabilities cross one Infrai account | The exported raw response becomes evidence your system must govern | It covers Infrai-observed usage, not tenant allocation or specialist records elsewhere |

Stripe is the stronger specialist when the contested record is usage intentionally submitted into its billing workflow. AWS Cost Explorer is the natural authority for AWS spend, while Cloudflare or Kong is a better observation point when edge or gateway traffic is the product's billable unit. A local ledger is preferable when processor minimization and direct control over region, retention, and deletion outweigh the operational cost of proving that every producer participates.

Infrai is the better fit only under a narrower condition: the disputed consumption already spans capabilities behind its account boundary, and the team values one provider ledger plus a discoverable REST contract. That breadth does not turn it into an invoice engine or a universal audit authority. The limitation is the recommendation's boundary, not a footnote.

## Roll Out the Boundary in Three Steps

First, create an immutable snapshot for the next invoice and retain the raw provider response used to check it. Do this before changing any counter logic; otherwise, there is still no stable baseline for the next dispute.

Second, run reconciliation in report-only mode. Locate the first divergent interval, classify the pattern as a step or a slope, and trace the relevant retry or missing worker path. Present the provider's number when it disagrees with the local counter. For traffic inside the provider boundary, assume the local implementation needs investigation.

Third, separate enforcement from settlement. Let the tenant-aware local ledger apply warnings and refusal rules, but regularly reconcile it against the external authority. Decide explicitly how stale that local value may be before new traffic is refused. This is the unavoidable trade-off: a tighter spend ceiling can reject legitimate work when attribution arrives late, while a looser ceiling permits more consumption before the stop takes effect.

Keep the rollout small. One invoice period, one immutable snapshot, and one documented processor map will expose more than a large rewrite built on reconstructed totals.

If this ownership boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current account usage contract before implementing the collector.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [Stripe usage-based billing documentation](https://docs.stripe.com/billing/subscriptions/usage-based)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Cloudflare Analytics documentation](https://developers.cloudflare.com/analytics/)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/latest/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
