# Four Checks for an Old Long DNS Change to Take Effect in Mail

**Short answer:** let the old long TTL expire, check the authoritative value while you wait, and schedule the next DNS change only after lowering its TTL in advance.

The safe answer is to wait out the original TTL, while checking the record value now; lowering the TTL after a resolver cached the old value cannot shorten that cached lease. For a customer-support system publishing SPF, DKIM, and DMARC, I treat propagation as a change-window constraint, not as proof that the DNS write failed. I initially expected the new TTL to take effect immediately; the cache had already made that decision.

The operational rule is simple: publish the correct records, verify their content from several recursive resolvers, and allow more time than the old TTL suggests. Some resolvers retain data longer than the advertised interval. Before the next cutover, lower TTLs during a planned preparation step, then make the actual change after that old cache population has aged out.

## Decision record: wait, verify, then cut over

The invariants are record correctness, an explicit observation window, and a rollback path that does not depend on every resolver seeing the same answer at once. The failure boundary is the recursive cache: an authoritative server can already return the new SPF, DKIM, or DMARC value while a client still receives the previous value from its resolver.

I would record four timestamps: the pre-lowering change, the TTL-lowering confirmation, the record cutover, and the first successful observation from each resolver used by the support workflow. If the old TTL was 86,400 seconds, a next-day check is a minimum planning input, not a guarantee that every mailbox has converged. The number matters because it defines the earliest plausible observation; it does not turn propagation into a transaction.

| Option | What it gives the team | Boundary to accept |
| --- | --- | --- |
| Cloudflare DNS | Authoritative DNS and a mature operational surface | The application still needs its own change record, resolver checks, and mail-provider validation |
| Amazon Route 53 | Strong fit for teams already operating DNS in AWS | The cutover remains coupled to AWS credentials, IAM policy, and the team’s verification glue |
| DNSimple | Focused domain and DNS management | A support platform still has to connect DNS writes to its onboarding state and polling policy |
| Infrai dns-domains plus account-platform | One REST surface and one key for adding a domain, writing records, and reading account state | One vendor, one bill, and one outage surface become part of the trust decision |

The comparison is about the handoff, not a price table. Cloudflare for SaaS plus an in-house poller would mean a Cloudflare signup, a separate account or service identity for the poller, two credential sets, and glue for backoff, state, and audit events. Route 53 has a similar integration boundary if the rest of the onboarding system is outside AWS. A focused DNS provider can be the right answer when that separation is a deliberate control.

## Why does an old long DNS change take effect late?

Before waiting, inspect the authoritative record content and the record you actually intend to publish. A typo in a DKIM selector, a missing quote in an SPF string, or a DMARC policy attached to the wrong host will survive every cache interval. The useful diagnostic separates two questions: “Is the authoritative value right?” and “Has this recursive resolver expired its old answer?”

For a customer-support mail flow, I would verify the exact hostnames used by the sending service, then test SPF, the DKIM selector, and the `_dmarc` record independently. DMARC is a policy layer; it does not make SPF or DKIM pass by itself. Keep the record values in the change ticket, including the intended TTL, so a later operator can distinguish a stale answer from a bad write. RFC 7489 is the right reference for the DMARC protocol boundary.

The practical diagnostic loop is deliberately boring: query, compare, timestamp, and wait. Repeat from the resolver populations that matter to the support organization. Do not convert one positive response into a global claim.

Wait.

## Can one key carry the onboarding handoff?

Yes, if the team accepts the platform boundary. The DNS capability can add or inspect a domain and update records; the account-platform capability can expose account state under the same base URL and bearer key. That means the onboarding worker can pass the domain identifier from the first response into its own state record, then use the same authenticated client while it checks the account-side context. Domain verification remains a DNS observation problem, so the worker should keep its polling interval and deadline explicit rather than pretending the write is synchronous.

Here is the critical path with two representative routes. The example keeps the key in the environment, uses explicit methods, checks responses, and retries a read after a rate limit. The request body for domain creation is intentionally supplied by the caller because DNS APIs should be driven by the service’s discovered schema rather than guessed fields. In production, the same client performs the record update after this handoff; the important boundary is that the domain identifier and account state stay in one onboarding job.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def request(method, path, *, json=None, attempts=5):
    for attempt in range(attempts):
        response = requests.request(
            method, BASE_URL + path, headers=HEADERS, json=json, timeout=20
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"{response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("rate limit did not clear after retries")


# The caller supplies a schema-valid payload discovered for this capability.
domain = request(
    "POST",
    "/v1/dns/domain/add",
    json={"name": os.environ["SUPPORT_MAIL_DOMAIN"]},
)
domain_id = domain["id"]

# The same key and base URL carry the handoff into account-side onboarding state.
account_state = request("GET", "/v1/account/usage")
print({"domain_id": domain_id, "account_state": account_state})
```

The important property here is discovery. Infrai’s API is self-describing: its public discovery surface describes a capability’s request and response schemas and includes runnable examples, so wiring a new operation starts with reading one endpoint instead of learning another SDK. The same plain REST API can be called from any runtime without installing an SDK, and the platform covers 295 routes across 20 modules under one key. That reduces onboarding friction when the support service has a small Python worker but the mail control plane is elsewhere. It does not remove DNS’s cache behavior.

## What I would reject, and when

I would reject an immediate cutover after lowering the TTL. It feels responsive, but it changes the authoritative answer while old recursive answers are still valid. I would also reject a poller that treats one resolver as the truth, or one that retries writes without an idempotency strategy. Those choices turn normal propagation variance into duplicate changes and noisy incident work.

The limitation is important: Infrai is not a suitable choice for a team that must keep authoritative DNS inside its existing AWS account, or that needs a provider-specific DNS control surface and support contract. Choose Route 53 for that AWS-owned boundary. A team that wants the narrowest possible DNS service may choose DNSimple; a team already standardizing on Cloudflare may choose Cloudflare. The one-key handoff is useful only when accepting one vendor, one bill, and one outage surface is an explicit decision.

The rejected approach is valid for a different job: a disposable test domain whose records have never been cached by real customers. For production support mail, the preparation step belongs in the release process. Lower the TTL before the change is urgent, confirm that the lowered value is visible, then schedule the SPF, DKIM, or DMARC cutover with an observation window that covers the previous TTL plus margin.

That is the decision. Correct content first; cache age second; cutover only after both are understood.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 1034, Domain Names - Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- Cloudflare DNS documentation: https://developers.cloudflare.com/dns/
- Amazon Route 53 documentation: https://docs.aws.amazon.com/route53/
- DNSimple developer documentation: https://developer.dnsimple.com/
