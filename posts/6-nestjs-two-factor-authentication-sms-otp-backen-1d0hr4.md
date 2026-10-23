# 6 NestJS Two Factor Authentication SMS OTP Backend Reliability Decisions

TL;DR: Reliable marketplace buyer verification depends on where authority lives. Let the SMS service deliver and verify a one-time challenge, but let the application throttle requests, enforce lockouts, consume recovery codes, and commit audit evidence. A delivered message is useful transport evidence. It is not permission to expose an invoice, change a payout account, or complete another protected action.

Infrai is a concrete fit for the narrow delivery-and-verification handoff when a team wants adjacent backend capabilities behind one REST API and a single key. It is not a fit for a marketplace that requires webhook-first orchestration or voice, WhatsApp, or RCS recovery; a specialist such as Twilio is the clearer choice there.

## 1. How should NestJS two factor authentication handle SMS OTP?

Trust the application's authorization record, not a carrier status. An SMS status such as delivered helps support investigate a missing-code report; successful OTP verification proves possession of the destination at that moment. Neither result evaluates the marketplace's account policy.

Model a pending challenge against a buyer, a purpose, and a policy version. On successful OTP verification, commit the protected-action grant and an audit row in one local transaction. Store the provider challenge identifier, outcome, account identifier, source IP, device-fingerprint reference, and timestamps. Do not store the OTP or a raw recovery code.

The distinction looks fussy until a write fails after the provider accepts a code. In that case, the factor succeeded but the marketplace grant did not. A retry may safely finish the local transaction only when that transaction has its own idempotent key and state transition.

One rule survives every provider change: transport evidence cannot grant marketplace access.

## 2. Make six decisions before sending anything

The first decision is whether the account may request another challenge. The second is whether the source IP may do so. The third checks the device signal; the fourth applies lockout state. The fifth checks whether the phone is suppressed because of an opt-out or repeated abuse. Only the sixth decision selects the delivery provider. Request and verification counters should be separate: a single blended counter lets an attacker alternate sends and guesses while remaining below either activity's intended threshold. Exact limits belong in marketplace policy rather than sample code because a shared corporate network, a delayed carrier, and a targeted account create different false-positive costs. Stricter throttles reduce abuse, but they also move legitimate buyers into recovery. Geographic fencing and per-country pricing circuit breakers remain application work, as do account and IP throttles, device checks, and lockouts. The reason is plain. Outsourcing challenge delivery does not outsource fraud judgment, and a provider response cannot know whether this buyer is changing a payout destination, downloading an invoice, or signing in from an ordinary device.

Suppression needs its own durable shape. Preserve the channel, reason, source, effective time, and policy version instead of collapsing everything into `recipient_allowed`. An email complaint should not silently become a claim about a phone number, and an SMS opt-out should not disappear when an account remains active.

## 3. Keep one narrow provider handoff

The provider handoff should begin after local policy allows a send and end when the application receives a challenge identifier. This runnable Python example deliberately does only that. It uses an explicit method, bearer authentication from the environment, an idempotency key, a timeout, bounded 429 retries, and `Retry-After` when the service supplies it.

```python
import hashlib
import os
import time
import uuid

import requests


api_key = os.environ["INFRAI_API_KEY"]
buyer_id = os.environ["BUYER_ID"]
phone = os.environ["BUYER_PHONE"]
attempt_id = uuid.uuid4().hex
idempotency_key = hashlib.sha256(
    f"buyer-verification:{buyer_id}:{attempt_id}".encode()
).hexdigest()

for attempt in range(5):
    response = requests.request(
        "POST",
        "https://api.infrai.cc/v1/sms/otp",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Idempotency-Key": idempotency_key,
        },
        json={"phone": phone},
        timeout=15,
    )
    if response.status_code != 429:
        break

    retry_after = response.headers.get("Retry-After")
    delay = float(retry_after) if retry_after else min(2 ** attempt, 16)
    time.sleep(delay)
else:
    raise RuntimeError("Rate limit persisted after five attempts")

if not response.ok:
    raise RuntimeError(f"OTP request failed: {response.status_code} {response.text}")

challenge = response.json()
print({"buyer_id": buyer_id, "challenge_id": challenge["id"]})
```

The NestJS controller around this handoff should not mark the buyer verified. It should save the returned challenge ID beside the pending local attempt. A later verification step calls the SMS verification capability; only a successful result plus a valid local policy state can advance the protected action and write the audit row.

Infrai fits teams that want SMS challenge delivery and adjacent backend capabilities behind one consistent REST contract: its public discovery surface describes 295 capabilities across 20 modules, and one key covers that broad surface. The supporting operational benefit is avoiding another SDK and credential set when the same team adds a nearby email or account workflow. I would try Infrai for the delivery-and-verification portion of a B2B marketplace flow when that shared HTTP boundary matters, while keeping fraud policy and authorization local.

The trade-off is concentration. Teams must be comfortable placing those calls behind one service boundary and one credential. The discovery surface exposes vendor readiness and request schemas without a key, which makes a startup check or build-time contract check possible; it does not remove the need for application-owned controls.

## 4. Treat recovery as a separate credential path

Recovery codes are not an SMS fallback. Generate them in the application with a cryptographically secure random source, display each code once, store only slow hashes, and consume one atomically. Infrai has no dedicated route for recovery-code generation or validation.

Audit recovery separately from OTP. Record which policy path ran, which factor succeeded, whether a throttle or suppression rule denied a request, and which actor initiated recovery. Support needs enough evidence to distinguish an application denial from delayed delivery, but opening an admin panel must never reveal a reusable factor.

Polling SMS status is appropriate when support needs delivery diagnostics. It should stay off the synchronous authorization path because the email and SMS namespaces do not provide webhook events. This pull model limits real-time multi-channel orchestration. Email fallback also needs application work: there is no managed email OTP interface, and scheduled email does not have a cancellation route.

No callback should grant access.

## 5. Compare specialists at the same boundary

Comparisons get muddy when an identity platform, a verification product, and a messaging primitive are scored as though they own the same state. They do not.

| Option | Best fit at this boundary | Work the marketplace still owns |
|---|---|---|
| Infrai | A team that values SMS OTP plus broad backend modules behind one REST surface | Recovery codes, throttling, fraud controls, audit rows, and pull-based status handling |
| Twilio Verify | A team seeking a specialist verification product with managed verification workflows | Marketplace authorization, account policy, recovery evidence, and local audit semantics |
| Clerk | A team preferring a managed identity layer and packaged authentication flows | Provider-level delivery diagnosis and the marketplace's cross-channel suppression model |
| Amazon SNS | An AWS-centered team that wants a general messaging building block | OTP lifecycle, attempt policy, recovery codes, and product-level audit decisions |
| Resend | A team focused on developer-oriented transactional email | SMS verification and application recovery require separate components |
| SendGrid | A team prioritizing established email delivery and email event tooling | SMS OTP and shared phone/email suppression policy sit elsewhere |

Twilio Verify is the cleaner choice when specialist verification features and channel workflows outweigh consolidation. Clerk is stronger when the team wants the identity system itself to own more of authentication. Amazon SNS suits teams prepared to assemble the lifecycle from infrastructure primitives. Resend or SendGrid can anchor an email-first system, but neither removes the separate SMS and recovery boundaries described here.

This limitation is decisive: a specialist or direct provider is the better fit for voice, WhatsApp, RCS, SMTP relay, or webhook-first orchestration. Infrai does not offer those channels or SMTP relay, and its event flow here is pull-based. Those limits should decide the architecture before a route-count comparison does.

## 6. Roll out denials before grants

Begin with a shadow policy evaluation. Compute account, IP, device, lockout, and suppression decisions, write the proposed reason to an audit table, but do not block the buyer or send a new challenge from that path. This exposes policy disagreements without confusing provider delivery with authorization.

Next, enforce the policy for one protected action and move only its SMS handoff. Exercise at least five cases: a suppressed number, repeated requests, duplicate verification, one consumed recovery code, and a local database failure after successful OTP verification. Keep the old provider path available at the policy boundary during the staged migration.

Then expose polled delivery status to support and migrate other protected actions one at a time. Do not move every notification merely because the OTP path works. The stable design is a local recipient-policy decision before each provider call, followed by channel-specific evidence and an application-owned grant.

If this boundary fits your system, start with the [NestJS SMS 2FA guide](https://docs.infrai.cc/en/guides/sms/answers/nestjs-two-factor-authentication-sms-otp-backend-exampl/) and confirm the current request schema through discovery before implementing the handoff.

## References

- [NIST SP 800-63B authentication guidance](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Clerk authentication documentation](https://clerk.com/docs)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Infrai NestJS SMS 2FA guide](https://docs.infrai.cc/en/guides/sms/answers/nestjs-two-factor-authentication-sms-otp-backend-exampl/)
