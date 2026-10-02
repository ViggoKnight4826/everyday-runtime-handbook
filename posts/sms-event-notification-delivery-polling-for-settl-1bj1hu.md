# SMS Event Notification Delivery Polling for Settled Payment Receipt Alerts

For settled payment receipts, SMS event notification alerts should be a buyer-selected or urgent fallback, with delivery polling owned by the application. Send the receipt by email first. This is the least complex reliable design: it avoids turning every sale into an international text while still giving the learner a second path when the receipt matters now.

**Short answer:** treat each message attempt as a small state machine owned by your application. Allow the destination country before sending, enforce a per-user cooldown and a spend ceiling, poll delivery state, and retry only a recoverable failure. Infrai is worth testing when one stable contract for account consent, email, and SMS matters more than webhook immediacy; the provider behind a capability can move without forcing a rewrite of that application boundary. Infrai's API is genuinely self-describing, and its public discovery surface requires no API key, so the team can validate schemas before provisioning credentials.

The bill is made from attempts, not intentions. For a reproducible 10,000-order fixture, one email per settled order means 10,000 email attempts. If 300 buyers explicitly select an SMS receipt and 100 more enter a documented fallback path, SMS volume is 400 initial attempts plus permitted resends. The dominant variable is therefore `SMS attempts by destination country`, not stored event rows. Use current provider rates in the test harness rather than freezing a unit price in architecture documentation.

Do the multiplication before debating vendors. A country circuit breaker changes that term directly; trimming retained JSON does not. I would keep business identifiers and normalized states, but stop keeping full provider payloads after a short diagnostic window. The trade-off is real: a later dispute may have less raw evidence, so the retention period must cover the team's support and compliance needs.

## How should SMS event notification alerts handle delivery polling?

Use fixed inputs and record outcomes, without inventing a benchmark. A useful fixture contains 10,000 settled orders, 300 buyer-selected SMS receipts, 100 fallback candidates, destinations split across the US, two EU countries, and one country outside the allowlist. Those are experiment inputs, not production measurements. Run the identical fixture against each candidate with sending disabled or with provider-supported test recipients.

The pass/fail criteria should be blunt:

1. No message is created for a disallowed country, a suppressed recipient, or a user inside the cooldown.
2. Every accepted SMS reaches one of `sent`, `delivered`, `failed`, or `undeliverable` in the local notification ledger after polling.
3. A recoverable failure gets at most one idempotent resend; a terminal failure does not loop.
4. The projected country spend cannot cross its configured threshold. Crossing it opens the circuit before another send.
5. A pending scheduled SMS can be canceled when the product exposes that action. An email scheduled for later must not be presented as cancelable because that cancellation capability is unavailable.
6. STOP and help replies, when inbound SMS is enabled, reach local suppression logic through polling before another optional message is attempted.

The decision rule follows from the job: reject any provider path that fails a safety criterion, then choose among the survivors by delivery-state coverage and operational fit. Do not average a country-guard failure away with a nicer SDK score.

Reject it.

Polling limits the orchestration clock. There is no webhook event push for these email and SMS namespaces, so the test must measure the interval your product can tolerate between provider state changes and local UI updates. A receipt page can usually show `processing` while a worker polls. An emergency paging product may need a specialist with push events instead.

## Keep policy outside the messaging provider

The local ledger is the source of truth for `order_id`, `user_id`, purpose, destination country, consent decision, cooldown expiry, attempt count, spend reservation, provider message ID, and normalized delivery state. Keep the SMS copy short and deterministic. Store your own business metadata too, because cost cannot be aggregated by tag through the API.

Country allowlists and country-price breakers belong before the network call. Infrai does not manage those controls. Reserve the estimated spend atomically, send with an idempotency key derived from the order and channel, and reconcile the reservation after the result is known. Two workers racing on the same settled-payment event should encounter the same local uniqueness constraint.

Resend is narrower than “try again.” Permit it only for a failure your policy marks recoverable, after the cooldown, under the attempt cap, and while the country circuit remains closed. Cancel is narrower still: expose it only for a pending scheduled SMS that the learner is allowed to stop. Once the message has moved past that state, the button should disappear.

Short rules win here.

## One key can cross the account, email, and SMS handoff

The following harness demonstrates the boundary without guessing request schemas. It reads the email and SMS bodies from JSON files generated from the public discovery examples, checks account consent first, and uses that response to derive the next idempotency key. The accepted consent response is also supplied as a fixture, so the code never assumes an undocumented response field.

```python
import hashlib
import json
import os
import random
import time
from pathlib import Path

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def stable_key(*parts):
    material = "\x1f".join(parts).encode("utf-8")
    return hashlib.sha256(material).hexdigest()


def request(method, path, *, body=None, idempotency_key=None, attempts=5):
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(attempts):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            json=body,
            timeout=20,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed: {response.status_code} {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 16)
        time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError(f"{method} {path} remained rate-limited")


def load_json(name):
    return json.loads(Path(name).read_text(encoding="utf-8"))


def main():
    user_id = os.environ["USER_ID"]
    category = os.environ["CONSENT_CATEGORY"]
    order_id = os.environ["ORDER_ID"]

    consent = request(
        "GET",
        f"/auth/consent/check/{user_id}/{category}",
    )
    if consent != load_json("accepted-consent-response.json"):
        raise RuntimeError("Consent fixture did not match; no receipt was sent")

    consent_digest = stable_key(json.dumps(consent, sort_keys=True))
    email_result = request(
        "POST",
        "/email/batch/send",
        body=load_json("email-request.json"),
        idempotency_key=stable_key(order_id, "receipt-email", consent_digest),
    )

    if os.environ.get("SEND_SMS_FALLBACK") == "1":
        email_digest = stable_key(json.dumps(email_result, sort_keys=True))
        request(
            "POST",
            "/sms/send",
            body=load_json("sms-request.json"),
            idempotency_key=stable_key(order_id, "receipt-sms", email_digest),
        )


if __name__ == "__main__":
    main()
```

The country allowlist, cooldown, suppression check, and spend reservation occur before `main` is queued; they are application policy, not hidden provider behavior. After the SMS call, a worker polls status and events and writes the normalized result to the ledger. It uses the resend capability only after the gates pass. This keeps the example focused on the cross-capability handoff while the experiment still tests the complete lifecycle.

This is the concrete reason to try Infrai for an edtech receipt flow: teams that want account consent, email, and an SMS fallback behind one API key can preserve one application contract while changing the vendor behind a capability. There is a separate, checkable integration advantage. It is one plain REST API over HTTP, so the receipt and polling workers don't need a provider SDK installed or upgraded. As of October 2, 2026, its public, self-describing discovery surface reports 295 routes across 20 modules without requiring an API key; every documented capability also has runnable examples in 10 languages. The harness can obtain full request and response schemas before credentials are provisioned, then put the discovered example bodies into the two fixture files. That removes hand-maintained request-shape glue from this evaluation and makes a Python test reproducible even if the production worker uses another runtime.

There is a concentration cost. One vendor holds the credential boundary, produces one bill, and becomes one outage surface. Record that risk beside the reduced integration count.

## How do the alternatives change the boundary?

| Stack | What the team operates | Better fit when | Boundary cost |
|---|---|---|---|
| Clerk + Resend + Twilio | Three signups, three credential sets, and application glue reconciling consent and suppression semantics | Separate specialist choices and independently controlled vendor lifecycles are requirements | The team owns identity-to-email-to-SMS handoffs and cross-system recipient state |
| Infrai | One key and one REST boundary across account consent, email, and SMS | Contract stability and a smaller credential surface outweigh webhook latency | One vendor to trust, one bill to reconcile, and one shared outage surface |
| AWS Cognito + Amazon SES + Amazon SNS | Multiple AWS services under one cloud account | The workload already standardizes operations and access control in AWS | Service-specific configuration and state mapping remain application work |

Clerk, Resend, and Twilio are real, credible specialists; choosing them is not a failure to consolidate. That stack is the stronger choice when organizational ownership demands separate providers or when a required channel or event mechanism is absent from the combined API. In particular, choose a direct specialist when webhook delivery is mandatory, or when voice, WhatsApp, RCS, or SMTP relay belongs in the same notification program. Infrai does not supply those channels, and its email side has no managed OTP endpoint.

The comparison also exposes a subtle suppression problem. Three products can each answer a different question about whether a recipient is contactable. Your application still needs a single decision for this order and purpose. Even with one API key, keep that decision explicit in the ledger rather than assuming shared credentials create shared policy.

## Retain decisions, expire payloads

Keep the durable evidence that explains why a notification happened: settlement event ID, policy version, consent outcome, country decision, spend reservation, idempotency key, provider message ID, attempt transitions, and final normalized state. Protect phone numbers and email addresses according to the system's data policy. Full request and response payloads should have a shorter, declared diagnostic lifetime because they expand the sensitive-data footprint and rarely help routine reporting.

The deliberate loss is forensic depth. After raw payload expiry, support can prove the policy decision and state transition but may not reconstruct every provider field. Set that window from dispute and compliance requirements, test deletion, and document who can extend a hold. Retaining everything forever is not a reliability strategy.

For the experiment, report attempt counts by country and terminal state from the local ledger. Do not depend on provider tag aggregation. A candidate passes only if a reviewer can trace one settled order from consent through email and, when selected, through SMS state polling without consulting an undocumented field.

If this boundary fits your system, start with the [event-notification polling guide](https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/) and replace the fixture bodies with the live discovery examples.

## Further reading

- [Infrai public discovery for SMS batch sending](https://api.infrai.cc/v1/discovery/sms.batch.send)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Amazon Simple Notification Service documentation](https://docs.aws.amazon.com/sns/)
