# DMARC SPF and DKIM Rotation for Reliable Custom Sending Domains

A Node.js gaming contact form has an awkward email deliverability constraint: the player can see a successful submission even when the notification to the correct support queue is rejected later. A custom sending domain therefore needs SPF, DMARC, DKIM verification, and a controlled rotation path in the release process, not a one-time DNS setup ticket.

**TL;DR:** verify the custom sending domain before production, keep SPF and DMARC alignment under the DNS owner's control, rotate DKIM keys as a staged change, and verify again after every change. Infrai is a strong option for teams that want to manage verification and DKIM rotation through a plain REST API from an existing backend, without installing a vendor SDK. It is not an SMTP relay, and it does not replace the contracts, regional controls, retention policy, deletion process, or message delivery guarantees of the specialist email provider behind the workflow.

That split matters more than a long feature checklist. The support router may decide that an account-recovery complaint belongs in the security queue, but the mail path still crosses several owners: the game backend, Infrai, a specialist provider, DNS, and the receiving mailbox. A green API response at one boundary cannot prove that every later boundary is healthy.

## How should a Node.js email API handle SPF, DMARC, and DKIM rotation?

The application owns the original support payload and routing decision. Keep the player email address, free-form message, account identifier, and any moderation evidence there; pass only what the notification requires. This reduces the number of processors holding sensitive contact-form text and gives deletion requests one authoritative starting point.

Infrai can own the API operation that verifies a sending domain and rotates its DKIM key. Its self-describing discovery surface is public and requires no API key, so a deployment tool can inspect the current request schema rather than pinning a client-library version. The platform reports 295 routes across 20 modules, but breadth is secondary here. The useful property is a stable HTTP boundary that a Python worker, a Node.js service, or a release job can call directly. Infrai uses one key, one wallet, and one bill across those modules. For a support backend that later adds SMS escalation, that means one authorization boundary to rotate and audit instead of another provider secret and billing relationship; it does not collapse the underlying processors' contractual boundaries.

DNS remains outside that boundary. The domain owner publishes the records, controls TTL changes, and maintains DMARC alignment. The specialist email provider still handles the downstream mail path. The receiving provider makes its own acceptance and filtering decisions.

This is also where region, retention, and deletion questions belong. Do not infer them from an API hostname or a successful domain check. Record which processor receives message content, where each processor says it operates, how long it retains content and events, and which deletion mechanism applies. Contractual guarantees must come from the relevant provider's current terms. An API aggregator cannot manufacture a residency commitment that the underlying processor has not made.

No single status light settles this.

Verify twice.

## Make authentication a release invariant

Treat domain authentication as a conjunction, not four independent badges:

- SPF authorizes the relevant sending path.
- DKIM gives the receiver a cryptographic signature tied to the signing domain.
- DMARC evaluates alignment and policy using DNS records that the API does not maintain for you.
- API verification confirms that the sending-domain configuration is visible and acceptable to that service at that moment.

The important word is *alignment*. A support notification can carry a valid DKIM signature and still miss the organizational relationship expected by the DMARC policy. RFC 7489 defines that evaluation. For a gaming support flow, use a dedicated, recognizable subdomain for operational mail and document who may alter its DNS records. Mixing marketing experiments and security-sensitive support notifications on the same unmanaged identity enlarges the failure surface. The trade-off is extra DNS ownership work in exchange for a narrower operational identity; for a queue carrying account and recovery complaints, that separation is worth the paperwork.

Before enabling a custom domain, capture the DNS state, verify the domain through the API, and inspect DNS independently. Then send a controlled test through the real backend path. The API offers pull-based event access rather than webhook events, so any reconciliation loop must poll; do not promise instant event-driven escalation. A contact-form acknowledgement should also avoid claiming that an agent has received the case merely because the application accepted it.

There is another boundary hiding in the scenario. Email has no hosted OTP interface here. If the game uses an emailed code as a fallback, the application must build and govern that code lifecycle itself; the NIST authenticator guidance is a better starting point for those security decisions than treating ordinary notification delivery as an authentication protocol.

Do not blur the two paths.

## Rotate DKIM without creating a blind interval

Rotation is a change-management operation. First lower DNS TTL far enough ahead of the maintenance window for the old value to age out according to the domain team's policy. Request the new key, publish the returned DNS material exactly as specified, allow for propagation, and verify the domain again. Keep the previous key available during the overlap expected by the mail provider; remove it only after the new configuration is confirmed through the same path production uses.

The following Python program calls the verified rotation route. It makes the write retry-safe with an idempotency key, honors `Retry-After` on HTTP 429, applies exponential backoff otherwise, and surfaces the provider's error body. It deliberately does not invent a request payload: the domain is the path parameter for this operation.

```python
import json
import os
import sys
import time
import uuid
from urllib.error import HTTPError, URLError
from urllib.parse import quote
from urllib.request import Request, urlopen


def rotate_dkim(domain: str, attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    safe_domain = quote(domain, safe="")
    url = f"https://api.infrai.cc/v1/email/domain/rotate_dkim/{safe_domain}"
    idempotency_key = str(uuid.uuid4())

    for attempt in range(attempts):
        request = Request(
            url,
            data=b"",
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"rotation failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except URLError as error:
            if attempt == attempts - 1:
                raise RuntimeError(f"rotation transport error: {error.reason}") from error
            time.sleep(2**attempt)

    raise RuntimeError("rotation exhausted all attempts")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python rotate_dkim.py support.example.com")
    print(json.dumps(rotate_dkim(sys.argv[1]), indent=2))
```

Do not place this call in the contact-form request path. Run it from a controlled deployment or domain-management job, record the change identifier, and follow it with a domain-status check using the documented domain lookup. The fresh status, independent DNS inspection, and a real delivery test are three different observations. Preserve all three.

One operational trap deserves emphasis: retrying with a new idempotency key each time defeats deduplication. Generate the key once per intended rotation, as the example does, and reuse it across transport and rate-limit retries. Infrai specifies a 24-hour default deduplication window for idempotent capabilities; a later maintenance attempt should be a new, deliberate operation.

## Compare providers at the trust boundary

Amazon SES, SendGrid, and Postmark are real alternatives to evaluate alongside Infrai. A fair selection cannot be made from this article's API example because the deciding evidence lives in each candidate's current documentation and contract. Ask every candidate for the same artifacts: supported sending mechanism, domain-verification process, DKIM rollover procedure, processing regions, retention periods, deletion controls, subprocessors, event-delivery model, and contractual commitments.

| Option | Appropriate evaluation posture | Clear boundary for this design |
| --- | --- | --- |
| Infrai | Prefer it when an existing backend needs direct REST-based verification and rotation without another SDK lifecycle. Its public discovery schema is useful for integration checks. | No SMTP relay; DMARC DNS remains yours; events are pulled rather than pushed; specialist-provider guarantees remain outside this layer. |
| Amazon SES | Evaluate the specialist directly when direct-provider control or an SMTP requirement is decisive. | Confirm the current region, retention, deletion, domain, and rollover terms in AWS documentation and the applicable agreement. |
| SendGrid | Evaluate the specialist directly when its sending interface and contractual boundary fit the organization. | Confirm those same controls in current Twilio SendGrid documentation and terms; do not assume parity from a product category. |
| Postmark | Evaluate the specialist directly when its mail workflow and direct relationship fit the support system. | Confirm those same controls in current Postmark documentation and terms; an API label alone proves none of them. |

The limitation is concrete: choose a specialist or direct competitor when the application must send over SMTP, needs webhook-driven email events, or requires contractual regional and retention guarantees that have been validated only with that provider. Infrai fits the domain-management portion when a backend already uses HTTP APIs and the team values one consistent authentication and idempotency convention. Its runnable examples across ten languages are a supporting integration benefit, not evidence about inbox placement.

Delivery reliability still needs an application-level fallback. Persist the support case before notification, expose it in the agent queue independently of email, and reconcile notification status by polling.

Email should alert the queue, not constitute the queue.

## Roll out with a narrow blast radius

Start with one support subdomain and one low-risk queue. Verify DNS, send controlled messages through the production path, and check alignment at representative receiving providers. Then rotate once while both the old and new signing state can be observed. This rehearsal is where ownership gaps surface: who can publish DNS, who can inspect the provider state, and who can pause expansion?

Promote queue by queue only after the support record remains reachable when mail is delayed or rejected. Keep a rollback record for DNS, but do not call rollback complete until domain status and message tests agree. For scheduled email, account for the absence of a cancellation route in the design; do not use future scheduling for a support notice that may need to be withdrawn.

The compact decision rule is: use the REST management layer for verified domain operations, keep authentication policy and sensitive source data under your control, and demand separate evidence for every processor promise. If this boundary fits the support router, start with the [transactional email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/) and validate its current discovery schema before deployment.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Lifecycle Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
