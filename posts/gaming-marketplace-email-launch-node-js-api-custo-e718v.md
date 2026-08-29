# Gaming Marketplace Email Launch: Node.js API, Custom Domain, DKIM, and SPF

Short answer: send the gaming marketplace's seller notification through an asynchronous transactional email API, authenticate a dedicated custom-domain subdomain with DKIM and SPF, and make the first email a controlled end-to-end test before enabling the order event in production.

The deciding constraint is integration effort. The Node.js checkout path should commit the order and a notification intent together; it shouldn't wait on DNS, a mail API, or a mailbox. A worker owns the external call, while delivery events update a separate state record. This keeps a delayed seller email visible without putting a completed marketplace order at risk.

This is an architecture decision record for one concrete message: when a buyer purchases a game item, notify the seller that order `ord_78421` needs attention. The same boundary can carry a welcome email, but the order notification makes the dangerous edge cases harder to ignore: duplicate sends can cause double fulfillment, stale links can expose the wrong order, and an open pixel is weak evidence that a seller acted.

## Who owns the domain release and notification policy?

Adopt a transactional outbox plus a stateless sending worker. The application writes an immutable event such as `marketplace.order.created`, the recipient account ID, the locale, and the template version in the same database transaction as the order. It does not store a pre-rendered authorization link. The worker later resolves the current allowed recipient address, renders a short-lived link, checks suppression, and submits the message with a stable idempotency key derived from the event and message purpose.

The key invariants are small enough to review:

1. A committed order produces at most one logical `seller-new-order` notification intent.
2. Email failure never rolls back or hides the order.
3. A retry reuses the same idempotency key and template version.
4. The visible `From` domain, return-path arrangement, DKIM signing domain, and published SPF policy are reviewed as one authentication setup before traffic is enabled.
5. A bounce, complaint, or explicit opt-out enters the suppression path before another nonessential message is attempted.
6. Logs contain internal IDs and state transitions, not access tokens or unnecessary message bodies.

There are three failure boundaries. Before submission, the service can reject bad application data without contacting the mail system. At submission, a rate limit or transport interruption can leave the outcome uncertain, so retry safety matters. After acceptance, mailbox delivery and user action are separate facts. Don't collapse them into a single `sent` boolean.

Acceptance isn't delivery.

DKIM is the cryptographic part of this boundary: RFC 6376 defines a domain-level signature that a receiver can verify using a public key published in DNS. SPF is a DNS authorization policy for the sending infrastructure. They improve authentication, but neither is consent, message relevance, or a promise of inbox placement. Keep those claims narrow.

Use two releases, not one. The domain release publishes the exact DNS records issued for the selected sending configuration and waits until the configuration reports verified. Use a dedicated transactional subdomain so order mail has an explicit operational boundary from employee mail and promotional campaigns. DNS changes should go through the same review and ownership process as other production configuration; copying a record into an unmanaged zone is quick, but discovering six months later that nobody owns its rotation is not.

DNS is deployment.

The application release creates the outbox producer and worker behind a disabled flag. In a staging-like environment, send the first email to controlled mailboxes that the team is authorized to use. Check the visible sender, reply behavior, text and HTML rendering, link destination, locale, and authentication results exposed by the received message. Then run a duplicate-event test: process the same outbox item twice and confirm that the system still has one logical notification. Run a suppression test too.

Only then enable a small production slice. Watch submission outcomes, delivery events, suppression growth, queue age, and time from order commit to API acceptance. Open rate does not belong in the release gate. Apple's Mail Privacy Protection can download remote content without the recipient engaging with the message, so opens cannot reliably prove that a seller saw an order. For this workflow, the marketplace's authenticated `viewed_order` or `accepted_order` event is the useful business signal.

Be strict about payload ownership. The worker should pass a recipient, a verified sender identity, a template version, and typed template data to a narrow gateway interface. Provider-specific response fields stop at that adapter. This makes a Node.js API migration an adapter change rather than a rewrite of checkout, policy, and notification state.

The first production send is still a test, just one with real consequences. Keep the cohort reversible, retain the outbox row, and make queue age alertable. Fast is good. Observable is better.

The table compares architecture, not brands. Actual feature support must be verified against the chosen service and contract during implementation.

| Integration shape | Application work | Operational work | Best fit | Limitation |
|---|---|---|---|---|
| Direct API call in the request | One client call and immediate response handling | Few moving parts initially | A disposable internal notification where loss is acceptable | Couples checkout latency to an external system and leaves uncertain outcomes awkward to reconcile |
| Database outbox plus API worker | Outbox schema, worker, idempotency, and event ingestion | Queue-age alerts, replay tools, and suppression ownership | Revenue-related seller mail where the order must survive independently | More code and state than a direct call |
| Managed workflow or campaign system | Event mapping, template data contract, and identity integration | Another workflow state model and permission surface | Multi-step lifecycle messaging owned by operations or marketing | Can duplicate product state and may be excessive for one deterministic order notice |

For this marketplace, the middle option wins because integration effort must include incident effort. Consider the awkward case — `ord_78421` commits at 14:03:18, the worker claims its notification, and the API connection closes without a readable response. The order exists and the remote acceptance state is unknown. If the checkout handler owns the call, a buyer retry may construct a fresh request while an operator has no durable notification record to inspect. With an outbox, the original event ID, purpose, claim, attempt, and idempotency key remain connected. The worker can place that item in an uncertain state, reconcile it according to the selected API's documented contract, and keep unrelated orders moving. If an event later reports a bounce, suppression attaches to the seller account before another optional message is selected. None of this guarantees an inbox placement; it gives each uncertainty one owner and one audit trail. That is why the apparently larger integration is easier to operate when fulfillment matters.

The catch is real. An outbox worker is not suitable when the team cannot operate background processing, event replay, and suppression correctly. For a low-risk internal tool, stick with a direct API call and accept the documented loss boundary. For a complex onboarding journey with branching delays and content ownership outside engineering, use a workflow system, but keep order truth and authorization in the marketplace database.

## How can a Node.js API implement a first welcome email on a custom domain with DKIM and SPF?

The production service is Node.js; the following Python is an executable model of the language-neutral contract. It focuses on the state transitions that need tests. The concrete HTTP adapter belongs behind `EmailGateway`, where authentication headers, timeouts, and the selected API schema can change without leaking into order logic.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Protocol


class SendResult(Enum):
    ACCEPTED = "accepted"
    RETRY_LATER = "retry_later"
    REJECTED = "rejected"


@dataclass(frozen=True)
class NotificationIntent:
    event_id: str
    order_id: str
    seller_id: str
    purpose: str
    template_version: str

    @property
    def idempotency_key(self) -> str:
        return f"{self.event_id}:{self.purpose}"


class EmailGateway(Protocol):
    def send(
        self,
        *,
        recipient: str,
        template_version: str,
        template_data: dict[str, str],
        idempotency_key: str,
    ) -> SendResult: ...


class NotificationStore(Protocol):
    def is_suppressed(self, seller_id: str) -> bool: ...
    def recipient_for(self, seller_id: str) -> str: ...
    def claim(self, idempotency_key: str) -> bool: ...
    def mark_accepted(self, idempotency_key: str) -> None: ...
    def release_for_retry(self, idempotency_key: str) -> None: ...
    def mark_rejected(self, idempotency_key: str) -> None: ...


def process_notification(
    intent: NotificationIntent,
    store: NotificationStore,
    gateway: EmailGateway,
) -> str:
    if store.is_suppressed(intent.seller_id):
        return "suppressed"

    if not store.claim(intent.idempotency_key):
        return "already_claimed"

    result = gateway.send(
        recipient=store.recipient_for(intent.seller_id),
        template_version=intent.template_version,
        template_data={"order_id": intent.order_id},
        idempotency_key=intent.idempotency_key,
    )

    if result is SendResult.ACCEPTED:
        store.mark_accepted(intent.idempotency_key)
        return "accepted"
    if result is SendResult.RETRY_LATER:
        store.release_for_retry(intent.idempotency_key)
        return "retry_later"

    store.mark_rejected(intent.idempotency_key)
    return "rejected"
```

`claim` needs atomic semantics. A process crash after remote acceptance but before `mark_accepted` is the sharp edge: the retry must carry the same key, and the adapter's documented idempotency behavior must be tested rather than assumed. If the selected API offers no idempotency contract, enforce one logical claim locally and give uncertain submissions a reconciliation state instead of immediately resending.

Treat responses by class. A rate-limit response such as `429` is retryable after the indicated delay; malformed recipient or template data is not. Cap attempts, add jitter, and alert on old work rather than spinning forever. I've learned to distrust a green submission metric when queue age is climbing. I'm not sure what queue-age threshold is right for every marketplace because seller expectations and fulfillment windows vary; an explicit service objective and observed fulfillment data should settle it.

Delivery events require the same discipline. Authenticate event ingestion using the mechanism documented by the selected service, store the raw event identifier, and apply transitions monotonically so a late `delivered` observation cannot erase a later bounce or complaint. A dashboard should separate API acceptance, mailbox delivery state, and marketplace action. Three states. Three different questions.

## Which failure boundaries rule out inline sending?

The rejected option is sending inline from the order-creation handler. It remains valid for an internal, low-volume tool where a missed notification has no financial or security consequence, the caller can tolerate mail latency, and duplicate submission is harmless. Those conditions do not hold for a gaming marketplace seller who may fulfill a scarce item after reading the message.

This decision should be revisited if the notification becomes a multi-channel, multi-step workflow; if legal or regional requirements change the allowed data path; or if measured operations show that the outbox burden exceeds the risk it controls. Revisit with evidence from queue age, duplicate claims, delivery state, suppression, and seller action. Don't use open pixels as the deciding evidence.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
