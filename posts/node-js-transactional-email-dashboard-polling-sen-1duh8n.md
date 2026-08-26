# Node.js Transactional Email Dashboard: Polling Sent, Delivered, Bounced by ID

The compliance constraint changes the design: a Node.js SaaS dashboard should poll delivery events in a worker, join every observation to a stored message ID, and show the observation time beside `sent`, `delivered`, or `bounced`. The browser should read that local record, not call a mail system on every refresh. This gives support a traceable answer without pretending that polling is an instant event stream.

**Short answer:** store the outbound message ID before the contact form is routed, poll unresolved IDs on a schedule, preserve the raw response as evidence, and derive the display state separately. Measure freshness, unknown-state rate, retry volume, and audit completeness before copying the pattern into production.

## The experiment note: a local read model beats page polling

The tempting implementation is a small dashboard that asks the provider about each message whenever an operator opens a row. It looks wonderfully small in a notebook. In a shared SaaS, it creates a moving target: refreshes repeat reads, page latency depends on an outside service, and there is no durable record of what the system knew at a particular time.

The chosen design has three pieces. The contact-form handler writes a send record and its opaque message ID. A scheduled worker polls IDs that are still unresolved or recently submitted. The dashboard reads a tenant-scoped table of observations. Simple.

But the timestamps matter. `observed_at` means “the worker saw this response then,” not “the mailbox accepted it at that exact moment.” That distinction should stay visible in the UI, because a compliance reviewer or support lead may otherwise read a polling timestamp as a provider-side event timestamp; the two clocks answer different questions, and collapsing them makes later investigation harder.

For a media support queue, the useful first slice is narrow. A reporter's licensing question can route to editorial support, while a billing question can route to accounts; the delivery view should answer whether the notification was submitted and what later transport evidence was observed. It should not claim inbox placement, sender reputation, or complete campaign analytics.

Start with the join.

## How should a Node.js SaaS poll transactional email events by message ID?

Create a durable state machine before writing the chart. `sent` means the send path recorded a provider message ID. `delivered` and `bounced` mean the event adapter received documented values that map to those states. An unfamiliar value remains `unknown`; it must not be quietly converted into a failure. That distinction is the difference between an audit trail and a reassuring dashboard.

The worker needs an idempotent key such as `(tenant_id, message_id, observed_at)` or a provider event ID when one is documented. Store the raw payload, its retrieval time, the normalized state, and the adapter version. The adapter version is useful when a contract gains a new event value: old evidence stays intact while the interpretation can be reviewed.

The following Python example keeps the provider boundary deliberately generic. It shows the local part that can be tested without inventing a vendor route or response field. A Node.js service can use the same tables and state transitions; the transport adapter is the only part that should know the external contract.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Any


KNOWN_STATES = {"sent", "delivered", "bounced"}


@dataclass(frozen=True)
class Observation:
    tenant_id: str
    message_id: str
    state: str
    observed_at: str
    raw_event: dict[str, Any]
    adapter_version: str


def normalize_event(
    tenant_id: str,
    message_id: str,
    raw_event: dict[str, Any],
    adapter_version: str = "1",
) -> Observation:
    candidate = str(raw_event.get("status", "unknown")).lower()
    state = candidate if candidate in KNOWN_STATES else "unknown"
    observed_at = datetime.now(timezone.utc).isoformat()
    return Observation(
        tenant_id=tenant_id,
        message_id=message_id,
        state=state,
        observed_at=observed_at,
        raw_event=raw_event,
        adapter_version=adapter_version,
    )


def should_poll(state: str, last_checked_at: datetime | None) -> bool:
    if state in {"delivered", "bounced"}:
        return False
    if last_checked_at is None:
        return True
    age_seconds = (datetime.now(timezone.utc) - last_checked_at).total_seconds()
    return age_seconds >= 60
```

The `60` seconds here is a test fixture, not a deliverability promise. Production cadence should be configuration derived from provider limits, message age, support expectations, and the cost of stale evidence. I would test the worker with duplicate events, a delayed event, an unknown status, and a transient `429` response. A transient read problem must not rewrite the last confirmed state as `bounced`.

## What evidence makes the dashboard useful for compliance?

Compliance evidence is more than a green row. For each contact-form notification, retain the tenant, queue decision, message ID, recipient class, submission time, last checked time, normalized state, raw event, and the actor or job that changed the local record. Restrict access by tenant and role. Give exports stable identifiers so a reviewer can connect a support decision to the underlying observation without relying on a screenshot.

Google's Email sender guidelines make authentication, spam rates, and subscription behavior part of sender operations. A delivery dashboard can point to those controls, but it cannot substitute for them. It also cannot prove inbox placement: a delivered transport event and a message appearing in a primary inbox are different claims.

Keep the audit record append-oriented where practical. An updatable “current status” column is convenient for the UI, while an observation table preserves the sequence that led there. Both views can exist. The current view is for support; the history is for investigation.

Consider one concrete review: a contact form arrives, the router assigns it to editorial support, and the send path records message ID `m-17` before handing the notification to the mail transport. The worker later observes `delivered`, so the current view can show that state with the worker's timestamp. If a reviewer asks why the notification was associated with editorial support, the queue decision and the original routing inputs remain in the application record; if the reviewer asks what the delivery system reported, the raw event and adapter version answer that separate question. If a later poll returns an unfamiliar value, the history records it as `unknown` instead of rewriting the earlier evidence. If a retry is rate-limited, the worker records the attempt and preserves the last confirmed state. This separation is useful because one row is carrying several kinds of accountability: business routing, transport observation, and operational action. Putting all of them into a single mutable status field makes the screen shorter, but it makes the explanation weaker exactly when someone needs to reconstruct what happened.

## Failure modes and the metrics worth evaluating

The main failure modes are easy to name and easy to miss. A lost message ID breaks correlation. Direct browser polling multiplies reads with every refresh. A retry loop can amplify rate-limit pressure. A guessed status mapping turns a new event into false certainty. A dashboard without freshness metadata makes “not observed” look like “not sent.”

Measure these before declaring the experiment successful:

- correlation coverage: the share of send records with a stored message ID;
- freshness: elapsed time between the latest eligible event and its local observation;
- unknown-state rate: documented and unexplained values separated clearly;
- retry volume and `429` responses by tenant and worker;
- audit completeness: records with raw evidence, timestamps, and tenant scope;
- queue-routing accuracy for representative media contact-form categories.

The eval harness belongs beside the worker. Feed it fixtures for sent, delivered, bounced, unknown, duplicated, and delayed events. Then assert both the normalized state and the evidence retained. This is the notebook-to-prod step that prevents a polished chart from hiding a broken join.

Prompt cost is a separate concern, but the same discipline applies if an agent later classifies the contact form. Store the original text, the chosen queue, the model decision, and the evaluation result; do not ask a model to infer delivery state from prose when a transport event is available. Your mileage may vary on classification accuracy, but the audit boundary should remain stable.

## When is polling the wrong choice?

The catch is that polling is not suitable when support requires an immediate event stream, when provider retention is shorter than the investigation window, or when the team cannot operate a scheduler and an evidence store. Choose a documented push-event pipeline for that requirement. A faster refresh interval cannot turn a pull contract into push.

That is the trade-off.

Keep it boring.

The design is also a poor fit for a finance ledger or a complete deliverability product. It lacks a universal definition of inbox placement, and SMS needs its own accounting: Twilio's explanation of GSM-7 and UCS-2 shows why character encoding can change message segmentation. Keep email delivery events and SMS segment counts as related but different measurements.

The conclusion is intentionally modest: persist IDs, poll from a controlled worker, preserve raw observations, expose freshness, and evaluate the failure cases. That is enough for a simple SaaS support dashboard. It is not enough to claim perfect delivery or regulatory compliance by itself.

## Sources

- https://support.google.com/a/answer/81126
- https://www.twilio.com/docs/glossary/what-sms-character-limit
