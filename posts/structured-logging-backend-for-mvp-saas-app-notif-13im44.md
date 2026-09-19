# Structured Logging Backend for MVP SaaS App: Notification Cost Attribution by Request ID

A support notification may have one request ID but several delivery attempts. If those attempts are collapsed into a single "failed" log line, neither the engineer investigating a missed reply nor the person attributing delivery cost can tell which attempt consumed resources. Short answer: emit structured events for each attempt, preserve a stable notification ID across retries, and select a hosted logging backend by testing whether it can filter both identifiers and aggregate attempt-level cost fields within your retention and access constraints. Pino or Winston can produce the records; neither choice determines whether the backend can answer the question.

## How should an MVP SaaS app choose a structured logging backend for notification failures?

The HTTP request that schedules a support reply ends before a worker sends it. A webhook can report acceptance later, and a retry may run under a different request. Joining everything by the original request ID alone loses that chain. Assign a notification ID when the work is created and an attempt ID before each send. Carry the request ID as context, not as the primary identity of the delivery. A user ID may help locate related events, but a support agent should not need unrestricted access to every user's logs to investigate one ticket. For this MVP, a backend passes the first selection test only if an engineer can retrieve all attempts for one notification, then filter to a tenant before searching by user ID; broad text search across unscoped customer records is the wrong access pattern even when the search box appears convenient.

Retries change the billable unit.

Keep the event vocabulary precise: `queued`, `attempt_started`, `provider_accepted`, `provider_rejected`, and `delivery_confirmed` describe different observations. Acceptance is not delivery. A timeout is not proof that the provider did nothing; blindly retrying after one can produce duplicates. Store a stable idempotency key at the send boundary when the delivery interface supports one, and reconcile uncertain outcomes before charging an attempt to a failure category. The backend needs exact-match filtering for notification and attempt IDs, plus a time range; substring search over serialized JSON is a poor substitute for typed fields.

## What should an attempt record carry?

Make the unit of attribution explicit. One attempt record can carry `channel`, `notification_id`, `attempt_id`, `request_id`, `tenant_id`, `outcome`, `cost_units`, and `occurred_at`. `cost_units` is a locally defined accounting measure, not a claim about a provider's bill: document whether it counts send attempts, accepted messages, or invoiced units. Keep the field absent until the unit is known. That distinction prevents an accepted SMS and an unbilled validation failure from being reported as the same cost.

For example, the application can log a sanitized event at the point where an attempt reaches a terminal outcome:

```python
def attempt_event(notification, attempt, outcome, cost_units=None):
    event = {
        "event": "notification_attempt_finished",
        "notification_id": notification.id,
        "attempt_id": attempt.id,
        "request_id": notification.request_id,
        "tenant_id": notification.tenant_id,
        "channel": attempt.channel,
        "outcome": outcome,
    }
    if cost_units is not None:
        event["cost_units"] = cost_units
    return event
```

This is an event schema, not a promise that every outcome is terminal for the notification. A rejected first attempt may precede a successful second one. Log timestamps in the actual emitter using a consistent clock and record a provider reference only if it is safe to retain. Do not put message bodies, email addresses, phone numbers, OTP values, or raw provider responses into a general-purpose search index. Redaction at ingestion is too late if an earlier collector, buffer, or export already retained the secret. The OWASP logging guidance calls out both sensitive-data exclusion and the need to sanitize event data before recording it.

## How should the backend choice be tested?

Compare behavior using the same small synthetic dataset, not a price table. Generate two tenants, three notifications per tenant, and at least one notification with two attempts: a timeout followed by an accepted send. Query by request ID, by notification ID, and by a scoped tenant-and-user identifier. Check whether a late status event can be correlated without changing the original record, and whether an exact-match filter distinguishes IDs that share a prefix. Measure the delay from emission to search during a burst. If cost attribution needs a daily rollup, verify that numeric `cost_units` stays numeric throughout ingestion and export.

Then test the uncomfortable boundary: turn off log ingestion for a short window and confirm that delivery still works, that the application's buffer is bounded, and that dropped-log counts are visible elsewhere. A logging outage should not hold up a support reply. Verify retention, deletion, export format, access controls, and the total billable ingestion path, including repeated retries and verbose payloads. Hosted storage may reduce the work of operating an index, but it transfers questions about retention and query permissions to the service contract. A self-managed index gives more control and more operational responsibility. **Choose the backend that preserves your event model and required queries under those constraints.**

Do not infer delivery from acceptance.

The logger library is a separate decision. If an existing Node.js service emits structured records through Pino or Winston, keep its output boundary stable and validate the JSON in the collector. Changing a logger does not fix missing correlation fields. OpenTelemetry's log data model supplies common concepts such as timestamps, severity, attributes, and trace context; W3C Trace Context defines trace identifiers for distributed requests. Neither standard replaces a business-level notification ID for work that outlives a request.

## Roll out without losing the failure trail

Start with one notification path and validate records against a schema in tests. Include the timeout-then-retry case, a late delivery status, a duplicate callback, and a rejected attempt that should carry no asserted cost until accounting confirms it. Compare event counts with the worker's attempt ledger during rollout; sampling away failure events makes incident reconstruction and attribution unreliable. Only after those checks should the same schema cover the other channels.

Keep old and new fields side by side for a bounded migration period, then remove the obsolete ones after saved queries and access policies are updated. The final operational question is concrete: can an on-call engineer find every attempt for a support notification while a finance query attributes units to the right tenant without exposing message content? If the answer is yes, backend choice becomes an operational trade-off instead of a brand contest.

## References

The standards and security guidance used for the event model and data-handling boundaries are listed below.

## Sources

- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://www.w3.org/TR/trace-context/
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
