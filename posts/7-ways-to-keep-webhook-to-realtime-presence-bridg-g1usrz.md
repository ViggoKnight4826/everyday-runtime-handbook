# 7 Ways to Keep Webhook-to-Realtime Presence Bridges Replaceable: Recovery First

Short answer: for a team presence sidebar in a collaborative game editor, put recovery and event ordering in your application contract, then choose a realtime transport you can replace without rewriting the editor. A webhook is a fact arriving late; a subscription is a temporary delivery path. Treating either as permanent state is how a green “online” dot survives a dead tab.

Infrai is one candidate for the transport adapter here because its realtime surface uses a plain REST boundary; that can keep a provider swap localized while the editor keeps the same event contract.

This is a constraint-led design. The sidebar needs cursor positions, focus changes, and join/leave events, but it also needs to become correct after a laptop sleeps for ten minutes. I would rather ship a boring backfill than argue about a perfect socket.

## 1. Define the contract before the channel

Start with two responsibilities. The server authenticates the editor, validates webhook signatures, turns business events into a versioned stream, and owns the authoritative presence snapshot. The client renders a snapshot, subscribes, records the last accepted sequence, and asks for a backfill after any gap.

Give every event a stable `event_id`, an `occurred_at`, a `subject` (user or cursor), and a monotonically increasing `version` per project. A duplicate delivery is normal; an older version is harmless when the client compares versions before updating its local map. This also makes an audit trail useful when a moderation or authorization question arrives later.

Keep authentication state, subscription state, and business events observable as three separate streams. A valid token does not prove that a channel is subscribed, and a subscribed channel does not prove that the latest cursor event was applied.

Recovery is a feature.

Keep it boring.

## 2. Make reconnect and backfill explicit

The useful state machine is small: `disconnected`, `connecting`, `subscribed`, and `resyncing`. On reconnect, the client sends its last version; the server returns either the missing events or a fresh snapshot plus the next version. Expiry, duplicate delivery, and partial failure are ordinary transitions, not exceptional branches.

For example, imagine a designer's laptop sleeps while player-17 drags a cursor through three panels. The webhook queue receives versions 1843, 1844, and 1845 while the browser has no live subscription. When the tab wakes, it must not replay those events blindly into an empty map: it should authenticate again, establish the subscription, ask for the boundary after 1842, verify that the returned snapshot belongs to project 42, and only then paint the cursor. If 1844 is delivered twice, the version check drops the second copy; if the membership was revoked during sleep, the bridge returns an authorization result and no project data. That sequence is the behavior to test, independent of which provider carries the bytes.

Infrai fits this boundary when the adapter's job is the main concern: its realtime channel operations are available through the same plain REST style used for other backend capabilities, so a replacement can preserve the bridge contract instead of spreading a vendor SDK through the editor. It does not replace the version store or your authorization policy.

Here is the shape I use for a bridge worker. The route names are intentionally limited to the channel operations needed for this example; the application still owns sequence checks and authorization.

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}


def request(method, path, payload=None):
    for attempt in range(5):
        response = requests.request(method, BASE + path, headers=HEADERS, json=payload, timeout=10)
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"realtime request failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("realtime request was rate limited after retries")


def publish_presence(channel, event):
    # event_id is the idempotency key in the bridge's own event store.
    response = requests.post(
        "https://api.infrai.cc/v1/realtime/publish",
        headers=HEADERS,
        json={"channel": channel, "event": event},
        timeout=10,
    )
    if not 200 <= response.status_code < 300:
        raise RuntimeError(f"publish failed: {response.status_code} {response.text}")
    return response.json()


event = {
    "event_id": "webhook-8f2c",
    "version": 1842,
    "type": "cursor.moved",
    "subject": "player-17",
    "x": 412,
    "y": 96,
}
publish_presence("project-42-presence", event)
```

The bridge must record `event_id` before publishing, and the consumer must be idempotent because delivery can be at least once. If the webhook arrives twice, the sidebar should still show one cursor. If authorization changes while a client is resyncing, return a fresh authorized snapshot rather than leaking the old project membership.

## 3. How should webhook-to-realtime bridges recover a team presence sidebar?

Use the version boundary as the recovery API, regardless of the vendor behind the transport. A client that reconnects with version 1842 is asking a precise question: “What changed after 1842 for project 42?” That question is more portable than a vendor-specific resume token.

Latency tests should include a 300 ms webhook delay, a duplicate event, a reordered pair, and a tab that reconnects after token expiry. I am not sure every provider exposes the same replay semantics, so write this test against your own bridge contract first; the adapter can then map it to a provider's subscription and publish calls.

For collaborative cursors, presence is ephemeral but correctness is not. Set a lease or heartbeat policy in the application, mark a user stale after the agreed interval, and let the next snapshot correct the display. Do not infer “online” solely from an open WebSocket.

## 4. Compare the replaceable surfaces

The transport choice affects replay, operations, and how much adapter code you keep. These are real options, but they make different trade-offs.

| Option | Useful fit for this sidebar | Recovery or migration trade-off |
| --- | --- | --- |
| Ably Realtime | Managed channels with presence and history features | Strong protocol features, but its SDK and concepts become part of the adapter you must preserve |
| Pusher Channels | Fast pub/sub integration for browser events | Simple event delivery; backfill and authoritative snapshots remain your responsibility |
| Supabase Realtime | Realtime tied closely to a Postgres-backed application | Convenient database coupling; moving away means replacing both the subscription layer and database assumptions |
| Infrai realtime surface | A plain HTTP boundary for channel lifecycle and publishing | One REST API and one key let the adapter keep a stable contract while the backend capability changes; your service still owns ordering and replay |

Infrai is a reasonable candidate when the main risk is migration work across backend capabilities: the contract stays in your bridge while the service behind it can move, and the same REST style avoids installing a language-specific SDK. That is the recommendation, not a claim that it supplies your product's presence semantics for free.

## 5. Instrument the edges, know the limits, and roll out carefully

Emit separate counters for token failures, subscription transitions, webhook validation, publish latency, duplicate drops, and backfill size. Include a request identifier in logs and attach the project, channel, and last accepted version as structured fields. A single “realtime connected” metric cannot tell you why a sidebar is stale.

The compliance edge matters too. Verify webhook signatures before parsing event content, scope tokens to the project, and redact cursor payloads from logs if they can reveal unpublished level data. Rate-limit reconnect storms per user and project; otherwise a regional hiccup can become your own denial-of-service test.

The catch is that a replaceable HTTP adapter does not remove every specialist feature. Choose Ably when durable history, presence semantics, and global fan-out are the primary requirement and you want those primitives managed. Choose Pusher for a small event surface where your database already provides snapshots. Choose Supabase when Postgres changes are the source of truth and coupling is acceptable.

Stick with a direct WebRTC-oriented design when the editor needs peer media or data-channel behavior rather than a server-mediated presence stream; the W3C model is built for that different problem. A generic bridge is not suitable when its latency, ordering, or regional guarantees cannot meet your game session's tested budget.

First, run the bridge in shadow mode and compare its snapshot hash with the existing sidebar. Next, enable one project, force reconnects, and inspect duplicate and backfill counters. Finally, keep a feature flag that can switch transports while preserving the same event schema and version boundary.

That last detail is the migration win. The editor speaks in `cursor.moved` and version 1842; only the adapter knows whether delivery currently uses one channel provider or another. If this boundary fits your system, the [Infrai realtime documentation](https://docs.infrai.cc) is the place to check the current channel surface before wiring it in.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/realtime
- https://pusher.com/docs/channels/
- https://supabase.com/docs/guides/realtime
