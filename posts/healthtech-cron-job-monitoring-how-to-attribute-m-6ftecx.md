# Healthtech Cron Job Monitoring: How to Attribute Missed-Job Heartbeat Alerting Costs

A practical Healthchecks alternative for a beginner running cron job monitoring in a Node.js SaaS is an external heartbeat recorder with explicit tenant, region, and job ownership. Alert on a missing successful completion, not merely a thrown exception. For a healthtech import, keep the heartbeat payload free of patient data, calculate retention cost per owner before choosing a tool, and test US and EU alert delivery as separate paths.

The bill is mostly retained telemetry multiplied by time.

Start there.

For each signal class, estimate `events per day × average bytes per event × retention days`, then add query and notification charges if the chosen service bills for them. A worked planning case might use 240 imports per day, 700-byte completion records, and 30 days of retention: `240 × 700 × 30 = 5,040,000` retained bytes before storage overhead, replicas, or indexes. Those are example inputs, not a benchmark; replace all three with measurements from the SaaS.

This framing changes the buying question. The easiest setup is not the one with the shortest signup form. It is the one that can prove a scheduled import stopped, assign the resulting telemetry spend to the right tenant and region, and delete identifying data without dismantling the evidence needed for an incident review.

## How should a beginner set up Node.js SaaS cron job heartbeat alerting?

Treat the scheduled Node.js process and its monitor as separate failure domains. The job emits a start receipt, then a completion receipt only after the imported result passes validation. An external evaluator knows the expected interval and grace period. If no valid completion arrives by `expected_at + grace_seconds`, it creates one missed-job alert. The evaluator must not depend on the cron process being alive; otherwise a scheduler failure silences both the work and its alarm.

Silence is evidence.

Use a stable `tenant_id`, `job_id`, and coarse `region` such as `us` or `eu` for attribution. Do not put a patient name, email address, medical record number, source filename, or row contents in a heartbeat. A completion record needs operational evidence: when the run finished, whether its result was valid, how many bytes the record occupies, and which cost center owns it. Detailed diagnostic logs can live under a shorter retention rule and tighter access controls. GDPR Article 17 is a concrete reason to know where personal data resides and how erasure works; the safer design is to exclude that data from this signal in the first place.

The status vocabulary should stay tiny: `started`, `succeeded`, and `failed`. Only `succeeded` advances the heartbeat. A job that starts and exits unsuccessfully is visible, but it is not healthy. Nor is a process that finishes while producing zero expected results. For a scheduled clinical-directory import, validation might require a parseable file, a matching schema version, and a committed batch marker. The monitor needs the validation outcome, not the records themselves.

Keep the clock explicit. Store timestamps in UTC, define schedule ownership in configuration, and decide how daylight-saving changes affect the source schedule before production. For a job expected every hour with a 15-minute grace period, the evaluator checks for a valid success after the prior deadline; it does not infer health from a recent application deploy or from an open process. I'm not sure a single grace value will fit both regions until their actual import-duration distributions are measured. That measurement, rather than guesswork, should settle it.

## Measure the dominant retention term before selecting a service

Cost attribution fails when every event lands in one unlabelled bucket. It also fails when labels become an accidental copy of business data. A small ledger is enough: count accepted heartbeat bytes by tenant, region, job, and UTC billing period. Apply the service's documented ingestion, retention, query, and notification rates outside the event path, because rate cards change while event semantics should not.

| Cost input | Measure locally | Attribution key | Decision it supports |
| --- | --- | --- | --- |
| Heartbeat ingestion | Accepted encoded bytes | Tenant and job | Which scheduled import creates the base load |
| Retained heartbeat data | Byte-days by retention class | Region and cost center | Where a shorter retention window matters |
| Diagnostic logs | Indexed bytes and query volume | Incident or job | Whether verbose evidence should expire sooner |
| Alert delivery | Attempts by channel and region | On-call policy | Whether retries or duplicate channels dominate |

The dominant term often reveals itself only after the multiplication. Suppose completion records are small but debug payloads are 40 times larger and kept for the same period. Optimizing heartbeat count will barely move the total. Sample the encoded sizes at the collection boundary, not the in-memory object size, then repeat the calculation with the real retention classes. If indexes or replicas are billed separately, include those documented multipliers as distinct rows rather than hiding them in a rough storage estimate.

Do not send one alert per polling cycle. Persist an incident key such as `(tenant_id, job_id, expected_at)`, notify on the transition into the missed state, and close that incident when a later valid run arrives. This is event grouping at the operational layer. Error trackers also group related events through grouping and fingerprint mechanics, but an error group does not prove that a scheduler invoked a job; a silent absence produces no application exception to group.

One incident. One alert.

Alert delivery deserves its own cost and reliability line. I've seen rate limits and delivery gaps turn a technically correct notification flow into a false sense of coverage. An HTTP `429` from an email or SMS provider should enter a bounded retry policy and a separate delivery metric — it should not cause the monitor to manufacture another missed import incident. Use two channels only when the health impact justifies the extra attempts, and test the escalation policy in both US and EU routes. Don't put sensitive import context into the notification body.

## Implement the heartbeat and missed-job ledger

The following focused reference uses only the Python standard library. It represents the monitoring side of a Node.js SaaS: the application can POST equivalent JSON to a collector, while this executable example stores already-validated receipts in SQLite and evaluates deadlines. The sample timestamps and byte counts are fixtures so the result is reproducible.

```python
from __future__ import annotations

import json
import sqlite3
from dataclasses import asdict, dataclass
from datetime import datetime, timedelta, timezone

UTC = timezone.utc


@dataclass(frozen=True)
class Receipt:
    tenant_id: str
    job_id: str
    region: str
    status: str
    finished_at: str


def encoded_size(receipt: Receipt) -> int:
    payload = json.dumps(asdict(receipt), separators=(",", ":"), sort_keys=True)
    return len(payload.encode("utf-8"))


def record_receipt(db: sqlite3.Connection, receipt: Receipt) -> None:
    if receipt.status not in {"started", "succeeded", "failed"}:
        raise ValueError("invalid receipt status")
    if receipt.region not in {"us", "eu"}:
        raise ValueError("invalid region")
    datetime.fromisoformat(receipt.finished_at).astimezone(UTC)
    db.execute(
        """
        INSERT INTO receipts
            (tenant_id, job_id, region, status, finished_at, encoded_bytes)
        VALUES (?, ?, ?, ?, ?, ?)
        """,
        (*asdict(receipt).values(), encoded_size(receipt)),
    )
    db.commit()


def missed_jobs(
    db: sqlite3.Connection, now: datetime, grace: timedelta
) -> list[tuple[str, str, str]]:
    cutoff = (now - grace).astimezone(UTC).isoformat()
    return db.execute(
        """
        SELECT tenant_id, job_id, region
        FROM schedules AS s
        WHERE s.expected_at <= ?
          AND NOT EXISTS (
              SELECT 1 FROM receipts AS r
              WHERE r.tenant_id = s.tenant_id
                AND r.job_id = s.job_id
                AND r.region = s.region
                AND r.status = 'succeeded'
                AND r.finished_at >= s.window_start
          )
        """,
        (cutoff,),
    ).fetchall()


def retained_bytes_by_owner(db: sqlite3.Connection) -> list[tuple[str, str, int]]:
    return db.execute(
        """
        SELECT tenant_id, region, SUM(encoded_bytes)
        FROM receipts
        GROUP BY tenant_id, region
        ORDER BY tenant_id, region
        """
    ).fetchall()


db = sqlite3.connect(":memory:")
db.executescript(
    """
    CREATE TABLE receipts (
        tenant_id TEXT NOT NULL, job_id TEXT NOT NULL, region TEXT NOT NULL,
        status TEXT NOT NULL, finished_at TEXT NOT NULL, encoded_bytes INTEGER NOT NULL
    );
    CREATE TABLE schedules (
        tenant_id TEXT NOT NULL, job_id TEXT NOT NULL, region TEXT NOT NULL,
        window_start TEXT NOT NULL, expected_at TEXT NOT NULL
    );
    """
)
now = datetime(2026, 8, 18, 12, 30, tzinfo=UTC)
db.execute(
    "INSERT INTO schedules VALUES (?, ?, ?, ?, ?)",
    ("clinic-042", "directory-import", "eu",
     "2026-08-18T11:00:00+00:00", "2026-08-18T12:00:00+00:00"),
)
record_receipt(
    db,
    Receipt("clinic-042", "directory-import", "eu", "started",
            "2026-08-18T11:01:00+00:00"),
)
print("missed:", missed_jobs(db, now, timedelta(minutes=15)))
print("bytes by owner:", retained_bytes_by_owner(db))
```

Run it with `python monitor.py`. The scheduled import has only a `started` receipt, so the evaluator reports it as missed after the 15-minute grace period. Change the fixture to `succeeded` and the missed list becomes empty. That is the essential deployment test: kill the scheduled invocation before completion, wait through the grace window, and verify exactly one incident and its intended route. Then run a late success and verify closure without erasing the original timing evidence.

This model deliberately keeps schedules separate from receipts. A schedule defines what should happen; a receipt records what did happen. Put a unique constraint on the production incident key, authenticate the collection endpoint, reject oversized bodies, and authorize tenant access before exposing the ledger through an API. The example omits the HTTP server because framework-specific middleware would distract from the state transition that must be correct.

## Choose by failure behavior, data location, and operating burden

Evaluate a managed heartbeat service, an observability suite, and a self-hosted monitor with the same drills. Send a valid completion. Send only a start. Skip the job entirely. Deliver a success after the deadline. Repeat each case in the US and EU path, then confirm who receives one alert, who can acknowledge it, and what usage record supports chargeback. A demo dashboard is weak evidence; state transitions are the contract.

Test both paths.

The catch is operational ownership. A self-hosted monitor can give a team direct control over data location and retention, but it adds upgrades, backups, availability, and notification delivery to the same on-call workload. It is not suitable when the team cannot keep the monitor independent of the workloads it watches. A managed heartbeat service reduces that maintenance surface, while its region choices, deletion controls, grouping model, export path, and billing dimensions must match the healthtech data policy. An observability suite may fit teams that need traces, logs, errors, and heartbeat evidence in one investigation, but its indexed telemetry can be excessive for a small SaaS whose only question is whether six imports finished.

No category wins every case. Stick with a focused heartbeat service when silent missed runs are the main risk and low setup effort matters most. Choose a broader observability system when the team needs correlated application evidence and already operates its access and retention model. Self-host when data control and internal integration justify owning another critical service. The decision should survive a deleted tenant, a delayed EU import, a notification rate limit, and a month-end request to explain which tenant created which cost.

Keep successful heartbeat receipts for the shortest period that supports alert review, audit obligations, and cost reconciliation. Keep detailed failure evidence under a separate policy. What you deliberately stop retaining is row-level diagnostic context after its defined window; the cost is that an older incident may be explainable only from aggregate timing and incident metadata. Make that loss explicit before shortening retention.

## References

- Sentry, "Event Grouping and Fingerprints": https://docs.sentry.io/concepts/data-management/event-grouping/
- GDPR Article 17, "Right to Erasure": https://gdpr-info.eu/art-17-gdpr/
