# Marketplace Email Deliverability Service in 2026 (Custom Domain Bounce Management)

TL;DR: For marketplace compliance notices, the simplest custom domain email deliverability service setup is an API-only sender behind a small application-owned adapter, with the marketplace retaining template revisions and a compact evidence ledger. The bill begins with message submissions, then grows through webhook processing and retained records. Reduce the term you control: keep normalized evidence and hashes for the required period, while expiring duplicate webhook bodies after a short, documented diagnostic window.

Use a workload model before comparing services or SMTP alternatives. If a notice goes to 100,000 accounts, the starting workload is 100,000 submissions. If each submission later produces four events, the event pipeline sees 400,000 inputs. Those are scenario inputs, not vendor benchmarks. The useful equation is `total operating cost = submissions + event handling + retained evidence + engineering labor`; the change that moves the avoidable term is storing one durable notice record plus selected event digests instead of treating every raw callback as permanent evidence.

This choice has a cost during an investigation. Once raw bodies expire, an engineer can prove the normalized transition and its digest but cannot inspect every original field. Keep raw events long enough to cover replay diagnosis and signature-validation disputes, set that period with security and records owners, then delete them on schedule. Do not promise forensic detail you deliberately stopped retaining.

## What should a custom domain email deliverability service handle?

Start with the claim the marketplace may need to support: an approved notice revision was submitted for an intended account, and later delivery signals were processed without silently rewriting history. That claim does not require a permanent copy of every rendered body, transport callback, debug header, and log line.

A compact record can contain an opaque notice identifier, opaque recipient reference, immutable template revision, content hash, submission time, current normalized outcome, last event time, and the digest of each accepted event. Keep recipient lookup data behind a separate authorization boundary. Tags and webhook metadata spread readily into logs, so they should carry opaque references rather than marketplace profile fields. Bounce handling and suppression management belong beside that evidence, but their policy scope remains owned by the marketplace.

Short records are easier to query. They are also easier to misunderstand.

API acceptance proves only that a request was accepted by the sending service. It does not prove inbox placement or that a person read the notice. The evidence model should name states narrowly instead of collapsing `accepted`, `delayed`, `delivered`, and `failed` into a cheerful boolean. A late event must not erase an already recorded terminal result, and duplicate delivery must not create a second transition. The retention boundary belongs in the design review, not in a cleanup ticket. Counsel and records owners determine how long notice evidence is required; security determines access and encryption controls; engineering implements deletion for raw payloads, rendered content, and operational logs separately. One duration for every artifact is convenient, but it defeats data minimization. A common first assumption is that more retained payloads always produce stronger evidence. The correction is practical: redundant raw data increases exposure and indexing work without improving the specific claim once the verified transition, digest, and approved content revision are durable.

Keep that distinction sharp.

## Template ownership decides what the ledger can prove

The marketplace should own the source template, approval state, locale, and immutable revision. At submission time, render that revision and hash the exact content. The ledger then joins policy approval to the attempted send without depending on a mutable external template name.

This is an explicit trade-off. Application ownership adds renderer maintenance, preview fixtures, and deployment coordination. Remote template ownership can reduce payload size and let non-engineering teams edit copy, but a mutable alias weakens the evidence chain: the same alias may refer to different content at different times. If remote rendering is unavoidable, require an immutable revision identifier and record it beside the local approval record.

Keep the adapter narrow. Its input is the recipient reference, approved template revision, variables, and an idempotency key; its output is a provider-neutral submission identifier. SMTP is unnecessary for this boundary. An API-only interface makes request identity explicit, while asynchronous events handle later outcomes.

Suppressions need two layers. The delivery service can enforce its suppression state, while the marketplace keeps the policy decision that explains why another attempt was or was not made. A hard bounce, a temporary mailbox condition, and a complaint should not be forced into one permanent local category. Preserve the original reason code for diagnosis, normalize it into a small internal taxonomy, and make scope and duration reviewed policy choices.

## Normalize callbacks before they become audit data

Webhook callbacks can be duplicated and arrive out of order. Verify authenticity using the service's documented mechanism before accepting an event, enforce a replay policy, and place an inbox-level uniqueness constraint on the external event identifier. Perform deduplication and insertion in one database transaction; a read-then-write check races under concurrent delivery.

The reducer below illustrates the application boundary. The adapter supplies normalized values, so vendor payload fields never leak into the audit model.

```python
from dataclasses import dataclass, replace
from datetime import datetime
from enum import IntEnum


class Stage(IntEnum):
    SUBMITTED = 10
    ACCEPTED = 20
    DELAYED = 30
    DELIVERED = 40
    FAILED = 50


@dataclass(frozen=True)
class NoticeEvidence:
    notice_id: str
    template_revision: str
    content_sha256: str
    stage: Stage
    last_event_at: datetime
    reason_code: str | None = None


def reduce_event(record: NoticeEvidence, event: dict) -> NoticeEvidence:
    next_stage = Stage[event["stage"].upper()]
    occurred_at = datetime.fromisoformat(event["occurred_at"])

    # A late callback cannot replace evidence already treated as terminal.
    if record.stage in {Stage.DELIVERED, Stage.FAILED}:
        return record
    if next_stage < record.stage and next_stage != Stage.DELAYED:
        return record

    return replace(
        record,
        stage=next_stage,
        last_event_at=max(record.last_event_at, occurred_at),
        reason_code=event.get("reason_code") if next_stage == Stage.FAILED else None,
    )
```

This function is intentionally incomplete. Signature verification, event uniqueness, durable append, and transaction handling sit around it. The current row serves operational queries; an append-only sequence of verified digests supports reconstruction when chronology matters. A queue isolates notice submission from callback-consumer downtime, and queue age is more actionable than a raw count of received events.

Test the awkward orderings: the same event twice, `failed` before `accepted`, a stale replay, and overlapping callback-secret rotation. Also stop the consumer, allow events to accumulate, and confirm that recovery neither skips nor duplicates transitions. The sender should not wait for the audit consumer.

## Domain authentication is a separate acceptance test

Template evidence does not replace domain authentication. SPF authorizes hosts for the relevant envelope identity. RFC 7208 limits evaluation to 10 terms that cause DNS queries, so an SPF record assembled from many `include` and redirect paths can exceed the processing limit even when each addition looked harmless. Inventory authorized senders and test the complete record after every DNS change.

Require DKIM support and validate the resulting message for the marketplace's sending domain during evaluation. Record the authentication configuration revision used by the release, but do not mistake a configuration record for proof that a recipient received or read a message.

Run the selection exercise with controlled data: configure a delegated sending subdomain, validate SPF and DKIM, submit the same idempotency key twice, produce a documented test failure, replay one callback, and export suppression and event state. The candidate passes only if the team can distinguish submission from later outcomes and reconcile the export with its own ledger. Setup speed is secondary to an explainable chain from approved template revision to normalized event.

Email also has a channel boundary. NIST SP 800-63B says email must not be used for out-of-band authentication. A marketplace may send a compliance notice by email, but it should not quietly reuse that design as an email-based authenticator. Treat authentication messages, ordinary notifications, and legally significant notices as different policy classes even if they share transport infrastructure.

The decision is therefore narrow: choose an API-only service that exposes the authentication, event, suppression, and export capabilities this evidence model needs, but keep templates and policy under application control. Retain enough to substantiate the notice, expire raw duplicates deliberately, and document the information that will no longer be available during a later incident. That final limitation is part of the architecture.

## Further reading

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
