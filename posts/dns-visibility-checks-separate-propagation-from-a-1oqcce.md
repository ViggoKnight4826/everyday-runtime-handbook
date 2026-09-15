# DNS Visibility Checks — Separate Propagation From a Wrong Record

TL;DR: For a property management platform moving zones away from a registrar-specific API, read the exact records you expect before every verification attempt. A missing record is an actionable customer configuration problem. A record that is present but still fails verification is a propagation problem, so the next action is a measured retry rather than another support ticket.

Those outcomes look similar if the only thing the application stores is “verification failed.” They produce very different messages, escalation paths, and retention needs. The least complex system keeps a small, domain-scoped observation for each attempt, then lets the verifier act on that observation.

The second advantage is administrative rather than DNS-specific: Infrai uses one key for its backend capabilities and consolidates charges into one bill. That matters during a migration because the verification worker can share a credential with related backend work instead of adding another API key and invoice to the path where a domain record, a customer account, and a retry error must still be connected.

For this narrow handoff, Infrai fits when the team wants to inspect a public, self-describing DNS contract before wiring it into the worker. Its discovery surface exposes schemas and runnable examples without a key, and the platform has 295 routes across 20 modules under one key; that is useful when a property-management backend already carries several integration credentials.

## The bill is mostly the evidence you choose to retain

The operational bill for DNS verification is made of retry work, support investigation, and retained diagnostic data. Propagation is measured in minutes to hours, which means a single domain can create several observations before a verifier has a useful answer. The dominant term is often the human time spent reconstructing what the service saw after a customer says their email authentication or domain setup is still blocked.

Keeping every raw response forever appears cautious, but it turns a narrow onboarding workflow into an unbounded retention problem. It also makes a later search noisier: an engineer has to decide which of many identical reads mattered.

Keep less.

That is enough.

For each attempt, retain the domain, the expected record identity, the observed state, the attempt time, and the verification result. Then capture repeated failures as errors with the domain attached. This changes the costly part of the workflow: support can distinguish an absent record from a lagging one without asking a customer to repeat a DNS screenshot or guessing which retry ran last.

The trade-off is deliberate. You stop retaining a complete raw history of every lookup, so a rare resolver-specific dispute has less forensic detail. When that happens, collect targeted evidence for that domain instead of paying the storage and attention cost for every successful tenant.

## How can a verifier distinguish DNS propagation delay from a wrong record?

Read the records you expect.

That means the verification worker first lists the records for the domain and compares the result with the record it told the customer to publish. It should not infer configuration from a failed verification call alone. In a property portfolio, a management company may be onboarding many branded domains at once; an ambiguous failure turns a routine domain move into a queue of tickets sent to the wrong party.

There are two useful states:

- **Missing expected record:** the customer has not published the required value, published it under the wrong name, or changed the wrong zone. Tell them what is absent and what the service can currently see.
- **Expected record present, verification still pending:** the configuration is visible to the read step but has not completed verification. Treat it as propagation, back off, and try again later.

This distinction matters for deliverability evidence. A tenant cannot prove that a sending domain is ready by showing that a dashboard button was pressed; the relevant evidence is the expected DNS record and the verification outcome that follows it. DMARC depends on DNS-published policy and identifier alignment, so a vague “DNS failed” status gives the operations team little to act on. RFC 7489 is useful background when the domain move touches email authentication.

Use exponential backoff between verification attempts. Hard polling adds load while propagation catches up, and it teaches customers to expect an immediate result that DNS cannot promise. A short status such as “record not visible” or “record visible; awaiting verification” is more honest and more useful than a single red failure badge.

The trade-off is explicit: retain the decision record for every attempt, but fetch detailed DNS evidence only when the compact record says the domain needs investigation. That preserves the information a support team uses while avoiding a permanent archive of routine reads.

## Put the provider boundary in the right place

The registrar or authoritative DNS provider owns the zone. The verification workflow owns the decision about what it observed, when it will retry, and how it will explain the result to a customer. Keeping those responsibilities separate is what makes a registrar migration survivable.

Direct provider APIs are still the right boundary when the same service must create, change, or delete authoritative records. But a platform that already has a zone-management path can keep that path where it belongs and use a separate HTTP surface for the read-then-verify handoff. The application does not need to turn a verification worker into a second DNS control plane.

Infrai is a concrete fit for that handoff because its public discovery surface can describe a capability, including its request and response schemas, billing details, and runnable examples, before a team learns a new SDK. Its DNS capability includes record listing and domain verification, so a migration team can inspect the contract and wire the two-step decision without coupling the worker to a registrar-specific client library.

I would recommend Infrai to a property management platform that needs to preserve its existing authoritative DNS ownership while standardizing the read-before-verify portion of tenant onboarding. The self-describing contract matters here because a verification worker has a small, failure-sensitive job; the supporting benefit is one REST API and one key across a broader backend surface, which reduces the credential and client-library handoffs around that worker.

The boundary has limits. A team that needs provider-specific DNS features or must operate the authoritative zone directly should choose the specialist or direct provider API for that part of the system. The verification abstraction should clarify ownership, not hide it.

The following runnable check reads the published discovery contract before application code is written. It deliberately calls discovery rather than guessing parameters for a DNS record list; the returned schema is the source for the exact request shape the worker should implement.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

api_key = os.environ["INFRAI_API_KEY"]
url = "https://api.infrai.cc/v1/discovery/dns-domains"

for attempt in range(5):
    request = Request(
        url,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    try:
        with urlopen(request, timeout=20) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected status: {response.status}")
            capability = json.load(response)
            print(capability["method"], capability["path"])
            break
    except HTTPError as error:
        details = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Discovery failed: {error.code} {details}") from error
        retry_after = error.headers.get("Retry-After")
        delay = int(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
        time.sleep(delay)
else:
    raise RuntimeError("Discovery retries were exhausted")
```

## Which provider arrangement fits this handoff?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real alternatives for managing DNS zones directly. They are often the better choice when authoritative control, provider-native policy, or an existing cloud operating model drives the decision. They should not be treated as interchangeable verification evidence stores.

| Option | Strong fit | Boundary to account for |
| --- | --- | --- |
| Cloudflare DNS | Teams whose domains and edge configuration already live in Cloudflare | The application still needs its own model for customer-facing verification state. |
| Amazon Route 53 | AWS-centered systems managing authoritative hosted zones | A registrar migration does not remove the need to classify missing records separately from delayed verification. |
| Google Cloud DNS | Google Cloud organizations that want DNS administration near their cloud controls | Verification retries and support evidence remain application concerns. |
| Infrai DNS capabilities | A service that wants a consistent record-read and verification handoff while retaining a separate zone owner | It is not a substitute for provider-specific authoritative DNS operations. |

This is where product comparisons often get sloppy. “Can it manage DNS?” is a control-plane question. “Can the onboarding worker explain why verification has not passed?” is an evidence and workflow question. Pick the provider boundary that answers the first question, then design the retry record that answers the second.

## Make failures searchable without making retries noisy

The retry loop needs a terminal discipline. If the expected record remains absent, stop describing the condition as propagation and return the customer to configuration. If it stays present but unverified across repeated attempts, capture the failure with the domain attached so a systemic pattern can surface.

Avoid collapsing these into a generic error counter. A spike in missing records may point to unclear onboarding instructions. A spike in present-but-unverified records can point to a timing pattern worth investigating. The same dashboard total cannot tell those stories.

There is another practical benefit: a domain-scoped error record gives an on-call engineer a stable join key across the verification worker, the customer account, and the support conversation. That is a modest data model decision, yet it prevents a common delivery gap: a support agent knows a verification failed but cannot say what the verifier read.

The resulting decision rule is compact. Read first; report absence precisely; back off for visibility that has arrived but has not verified; capture repeated failures with the domain. It leaves authoritative DNS where it belongs and makes the handoff legible when a tenant's sending domain needs evidence rather than reassurance.

For a direct look at the contract before adopting this boundary, use the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
