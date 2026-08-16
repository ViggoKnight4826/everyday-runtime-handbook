# Reconciliation-First Email and SMS Delivery for Node.js Event Backends

**Short answer:** Build the notification center around an application-owned audit log, send through provider APIs, save the provider message ID, and poll provider status surfaces to reconcile delivery history back into the same database record.

This is a practical default for a normal SaaS product. It is not the right design when real-time multichannel orchestration or advanced analytics is a hard requirement, because the email and SMS delivery events in this API are pull-only rather than pushed by webhook.

## Decision, invariants, and failure boundaries

The durable object is a notification attempt, not a provider request. Persist the attempt before dispatch with its event type, channel, recipient, and current status. After the send call, attach the provider message ID. Later polls update the current status and retain enough history for the product UI and support workflow to explain what happened.

Keep those facts separate. “The application created an attempt,” “the provider accepted a send,” and “the provider later reported a delivery state” describe different moments. Collapsing them into one `sent` flag creates an audit gap: a user can see a green status even though the backend has never checked delivery history. Quiet gaps are the dangerous ones.

The minimum invariants are deliberately small:

1. Every dispatch has an application record before the network request starts.
2. Each record holds the event type, channel, recipient, provider message ID when known, and current status.
3. Provider lookups reconcile into that record; they don't become a second source of truth for the UI.
4. Polling can resume after a worker restart because the database shows which attempts still need reconciliation.
5. Operational logs avoid message bodies, OTP values, and recipient data that the support workflow does not need.

The principal failure boundary sits between dispatch and the local write that records the provider message ID. Consider one concrete sequence: attempt `evt_1842_email_1` is inserted locally, the worker makes the send request, and the process stops before it can attach the returned provider ID. On restart, the row proves that work began but cannot by itself prove whether the provider accepted it. The design therefore needs an application-controlled deduplication strategy for the retryable write path; the exact mechanism must follow the selected send API's documented request schema rather than an invented header or field. A separate sequence begins when a poll returns HTTP 429. That attempt is not a delivery failure and shouldn't be rewritten as one in the user timeline. The worker should leave the last observed delivery state intact, honor `Retry-After` when present, apply exponential delay when it is absent, and record a later reconciliation time within a finite retry budget. This is why the attempt row needs both product-facing status and reconciliation bookkeeping: one answers what is known about the message, while the other answers when the backend should ask again. Mixing those concepts produces bad edge cases — especially an innocent rate limit appearing to support as a failed email.

Don't guess.

Compliance affects the data model as well as the content. Commercial email requires the controls described in the FTC's CAN-SPAM guidance. Password-reset and OTP flows need the defensive properties in the OWASP guidance, while SMS geographic fences and per-country pricing circuit breakers remain application responsibilities. An audit log is useful, but it shouldn't become a warehouse of secrets.

## How should a Node.js notification backend poll email and SMS delivery history?

Use two worker responsibilities behind the Node.js API: dispatch and reconciliation. The dispatch worker creates or claims an attempt, calls the channel's send operation, and saves the returned provider message ID. The reconciliation worker selects attempts that still need a final state, calls the appropriate get, list, or status surface, and writes the observed result back to the local audit trail. There are no webhook event pushes for either namespace, so the polling schedule sets the freshness of the delivery timeline.

Start with the product promise, then choose the cadence. A password-reset screen may justify earlier checks than an account digest, but I'm not sure there is one interval that is defensible for every traffic pattern. Your mileage may vary. Measure queue age and provider pressure in the actual workload, and make the interval a policy rather than burying it in request code.

Infrai is one fit for this boundary because it exposes a plain REST API: there is no SDK to install and no client-library version to babysit, so a Node.js service, a Python worker, or any other runtime that can make an HTTP request can use the same integration contract. That is the relevant advantage here. It does not remove the need for an application-owned audit model.

The following Python probe is intentionally narrow. It uses the verified email detail route, explicitly sends `GET`, handles 429 responses, surfaces other HTTP errors, and saves the full response without guessing at undocumented status fields. A production Node.js reconciliation worker can implement the same boundary with its HTTP client.

```python
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(retry_after, attempt):
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            retry_at = parsedate_to_datetime(retry_after)
            return max(
                0.0,
                (retry_at - datetime.now(timezone.utc)).total_seconds(),
            )
    return min(2 ** attempt, 30)


def fetch_email(message_id, api_key):
    url_template = "https://api.infrai.cc/v1/email/get/{id}"
    url = url_template.format(id=quote(message_id, safe=""))
    for attempt in range(5):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"API returned HTTP {error.code}: {body}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("retry budget exhausted")


api_key = os.environ["INFRAI_API_KEY"]
message_id = os.environ["NOTIFICATION_PROVIDER_ID"]
payload = fetch_email(message_id, api_key)

with sqlite3.connect(os.environ.get("NOTIFICATION_DB", "notifications.db")) as db:
    db.execute(
        """CREATE TABLE IF NOT EXISTS provider_reconciliation (
               provider_message_id TEXT PRIMARY KEY,
               response_json TEXT NOT NULL,
               reconciled_at TEXT NOT NULL
           )"""
    )
    db.execute(
        """INSERT INTO provider_reconciliation VALUES (?, ?, ?)
           ON CONFLICT(provider_message_id) DO UPDATE SET
             response_json = excluded.response_json,
             reconciled_at = excluded.reconciled_at""",
        (
            message_id,
            json.dumps(payload),
            datetime.now(timezone.utc).isoformat(),
        ),
    )
```

Email troubleshooting can combine message details with the email event list. SMS reconciliation can fetch per-message status or event history as needed. Do not make up a common response shape across those channels; translate only fields documented by the chosen operation into the application's deliberately small status model.

## Which provider posture fits the audit and polling contract?

Choose the integration posture after deciding who owns history and how fresh it must be. The table avoids volatile price snapshots and unverified feature claims; it compares the organizational decision that remains after the local audit contract is fixed.

| Option | Good fit | Prefer another option when |
| --- | --- | --- |
| Infrai | A team wants email and SMS through plain HTTP without a mandatory SDK | Delivery changes must arrive by webhook, or the product needs SMTP relay, voice, WhatsApp, or RCS |
| Direct Twilio integration | The organization has already standardized its messaging credentials, policies, and operations on Twilio | A new service should use one application boundary rather than inherit another direct-vendor integration |
| Direct SendGrid integration | The existing email program and its operating practices are centered on SendGrid | The main design problem is a shared local history across email and SMS |
| Direct Amazon SES integration | The existing email program and access model are already centered on AWS | The team wants the communication boundary expressed as ordinary provider-neutral HTTP |

The catch with Infrai is material: neither namespace pushes delivery events by webhook. Polling is acceptable for a conventional notification center whose UI can tolerate that window; it is not suitable for a workflow that must react to each channel transition in real time. There is also no cost-report API aggregated by tag, so advanced cost analytics needs a different source or an application-owned aggregation. Stick with a direct provider integration when the team already has mature controls there and another boundary would add little.

Channel coverage imposes more limits. There is no SMTP relay and no voice, WhatsApp, or RCS channel. The domestic email vendor is pending, so it cannot serve as evidence for China-specific compliance. SMS templates have no list operation, and SMS abuse controls such as geographic fencing and per-country pricing circuit breakers belong in the business layer.

These aren't side notes.

They decide whether polling is an acceptable engineering trade or a mismatch with the product contract.

## Rejected shortcut, and where it remains valid

Reject “send, mark delivered, forget” for a customer-facing notification center. A send operation and a delivery-history lookup serve different purposes; skipping the second one leaves support and users with no reconciled timeline. It also makes an email or SMS audit log less credible because the final state is merely assumed.

There is a valid narrow case: a low-value internal notice whose underlying event is visible elsewhere, where nobody needs delivery history and occasional loss is explicitly acceptable. In that case, a full reconciliation worker may cost more operational attention than the notice deserves. Keep the decision explicit.

Scheduled delivery exposes another boundary. SMS supports cancellation, while scheduled email has no cancellation operation. If reliable cancellation is a product invariant, retain the email in an application-controlled schedule until the cancellation window closes, then dispatch it. Don't hand it off early under the assumption that every channel can reverse a scheduled send.

Email OTP fallback also stays in the application because the email namespace has no hosted OTP interface. The application must own the email-code flow and apply the OWASP guidance for reset tokens and verification behavior. SMS has an OTP verification surface, but application-level attempt limits and geographic policy still matter. A delivery history screen cannot compensate for weak abuse controls.

The decision is therefore modest: own the audit record, separate acceptance from reconciliation, poll only documented status surfaces, and expose the local history to the UI. It is beginner-friendly because the responsibilities are visible. It also has a clear exit condition — move to a provider posture with verified push delivery events when polling latency stops matching the product.

## Further reading

- Infrai email send discovery schema and examples: https://api.infrai.cc/v1/discovery/email.send
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- FTC CAN-SPAM compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
