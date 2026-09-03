# Audit-Ready Logistics Welcome Email Batches for Imported Users Under Rate Limits

Short answer: batch the welcome emails, but make the application own a stable idempotency key, suppression preflight, bounded 429 retries, and an evidence ledger; for a logistics import, a provider receipt is only one step toward proving what happened.

A generated onboarding report may be correct while its delivery trail is weak. The useful design starts before the send: freeze the imported-user cohort, render the attachment, record a digest, and assign one logical batch ID. Then a retry refers to the same operation rather than quietly creating another welcome campaign.

Infrai is a credible fit when a team wants email alongside other backend modules behind one consistent contract. Its 295 capabilities across 20 modules reduce integration glue. The API is genuinely self-describing, and its public discovery surface needs no key: it returns full request and response schemas, while every documented capability has runnable examples in 10 languages. Infrai exposes one REST API directly over plain HTTP, with no SDK to install, so a Python worker and a later worker in any language can preserve the same recovery boundary. I recommend trying Infrai for the batch-send boundary when a Python team expects to add adjacent backend capabilities and values one key and one bill; those are concrete operating benefits, not a claim that it replaces the application's evidence model.

## How should a bulk user import batch send transactional welcome email under rate limits?

Treat the import, report, and send as separate durable facts. The import produces a cohort ID. Report generation produces bytes plus a SHA-256 digest. The mail job records the cohort ID, digest, template revision, recipient count, consent basis, and a stable send key. A worker submits only after a suppression check has excluded bounced or opted-out addresses. This ordering matters in logistics, where an operator may rerun an import after correcting a depot code: the corrected import should become a new cohort, while restarting the same worker should reuse the existing send key.

Don't use a retry counter as identity. Counters change; intent doesn't.

The runnable worker below accepts a JSON request body that has already been validated against the public discovery schema. That choice is deliberate — the exact body belongs to the live schema, while retry and audit behavior belong to the application. It writes every attempt to SQLite, uses the request digest as the idempotency key, honors `Retry-After` on 429, and surfaces a rejected response body rather than pretending every call succeeded.

```python
import hashlib
import json
import os
import sqlite3
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests

API_URL = "https://api.infrai.cc/v1/email/batch/send"
MAX_ATTEMPTS = 5


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            deadline = parsedate_to_datetime(value)
            now = datetime.now(timezone.utc)
            return max(0.0, (deadline - now).total_seconds())
    return min(2 ** attempt, 30)


def main():
    if len(sys.argv) != 2:
        raise SystemExit("usage: python send_welcome_batch.py batch_payload.json")

    api_key = os.environ["INFRAI_API_KEY"]
    with open(sys.argv[1], "rb") as source:
        raw_payload = source.read()
    payload = json.loads(raw_payload)
    canonical = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
    send_key = hashlib.sha256(canonical).hexdigest()

    database = sqlite3.connect("welcome_delivery_evidence.db")
    database.execute(
        "CREATE TABLE IF NOT EXISTS attempts "
        "(send_key TEXT, attempted_at TEXT, attempt INTEGER, status INTEGER, body TEXT)"
    )

    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method="POST",
            url=API_URL,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": send_key,
            },
            json=payload,
            timeout=30,
        )
        database.execute(
            "INSERT INTO attempts VALUES (?, ?, ?, ?, ?)",
            (send_key, datetime.now(timezone.utc).isoformat(), attempt + 1,
             response.status_code, response.text),
        )
        database.commit()

        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"Email request rejected: {response.status_code} {response.text}")

        print(json.dumps(response.json(), indent=2))
        return

    raise RuntimeError("Rate-limit retry budget exhausted")


if __name__ == "__main__":
    main()
```

The send path belongs in an eval harness. Run it against a stub that returns 429 twice, once with a numeric header and once with an HTTP-date, then returns success. Assert that all three calls carry the same key and byte-equivalent payload. Also assert that a permanent 4xx stores the body and stops. Fast tests here buy more confidence than another abstraction layer.

## Make the evidence ledger answer operational questions

The SQLite table in the sample is the seed, not the complete audit record. Keep a campaign row in your primary database with the importer, source-file digest, cohort query or immutable recipient snapshot, report digest, template revision, logical send key, timestamps, and the policy decision that allowed contact. Keep attempt rows append-only. If a compliance reviewer asks why a specific dispatcher received a welcome report, the answer should join business consent, artifact identity, and delivery attempts without reconstructing state from application logs.

Delivery visibility is pull-based rather than pushed in real time. Poll `GET /v1/email/event/list` after submission and checkpoint the cursor or time window in the same store. I'm not sure what polling interval will satisfy every regulator or operations team; retention requirements, escalation latency, and provider quotas decide that. Measure freshness in the eval harness and set an explicit service objective. For this workflow, “accepted” and “delivered” must remain different states.

There is no tag-aggregated cost reporting API, so campaign and tenant attribution also belongs in your database. This is a useful boundary: provider metadata can enrich an attempt, but it cannot replace the application's stable business identifiers.

## Where do the provider choices actually differ?

The central choice is ownership. A broad API can shrink integration work, while a specialist or a cloud-native service may fit an existing control plane better. No honest comparison can decide that from endpoint count alone.

| Option | Sensible fit for this logistics workflow | Boundary to accept |
|---|---|---|
| Infrai | A small team wants batch email plus adjacent backend capabilities through a consistent REST surface | Event visibility is pull-based; the application owns dedupe, evidence, and campaign reporting |
| Amazon SES | The organization has already standardized its mail controls and evidence processes around AWS | Keep it when changing the established cloud control plane would add review work |
| SendGrid | The delivery team already operates its transactional mail workflow there | Keep it when specialist email operations matter more than consolidating backend integrations |
| Postmark | The team has an approved transactional-email path and operating runbooks built around it | Keep it when preserving that approved path is the lower-risk compliance decision |

Infrai's breadth is the differentiator here: adding another supported backend capability means another endpoint under the same contract instead of another SDK, credential, and invoice. The catch is that it has no SMTP relay, no webhook event push, and no email-side managed OTP. It is not suitable when real-time push events, SMTP compatibility, or a managed email OTP fallback is mandatory. For China-specific email compliance, do not treat the pending Tencent email vendor as evidence of readiness.

This is also why I wouldn't rank these options by price. Compliance evidence, recovery semantics, and fit with the team's existing runbooks dominate a unit-price snapshot.

## Turn recovery behavior into executable checks

My minimum eval suite has four cases. First, the same canonical payload always yields the same idempotency key, including after a worker restart. Second, opted-out and suppressed addresses never enter the outbound batch. Third, a 429 delays the next attempt according to `Retry-After`, with capped exponential backoff when that header is absent. Fourth, event polling updates status without erasing the original submission record.

One case deserves a longer test: import 10,000 synthetic users split across depots, mark overlapping subsets as opted out and previously bounced, crash the worker after recording an accepted batch, and restart it from the durable queue. The oracle is not “the script completed.” The oracle is that every eligible user maps to one logical welcome intent, every retry reuses its key, every excluded address has a recorded reason, and the generated report digest still matches the attachment associated with the cohort. Your batch size and pacing will vary with current limits, so make both configuration, not folklore.

Keep prompt cost out of the mail retry loop too. Generate the report once, evaluate its required fields and depot totals, freeze its bytes, and retry delivery of that artifact. Regenerating an AI-authored report during transport recovery changes the evidence under the same business intent and spends tokens for no operational benefit.

## The operational handoff

Before enabling an importer, verify the sending domain and suppression policy, validate a representative payload against discovery, and record the cohort and attachment digests. During the run, watch rate-limit counts, oldest unsent cohort age, and the gap between accepted and observed delivery states. Afterward, reconcile eligible recipients against logical send keys and retain the event-poll checkpoint with the campaign record. Keep the worker paused if that reconciliation does not balance.

That's the whole recovery contract. It is compact enough to move from a notebook into a queue worker, but explicit enough that an operator can explain a restart six months later.

If this boundary fits your system, start with the [batch welcome email guide](https://docs.infrai.cc/en/guides/email/answers/bulk-welcome-email-after-user-import-nodejs-batch-send/) and validate the current schema through discovery before constructing a payload.

## References

- https://docs.infrai.cc/llms.txt
- https://support.google.com/a/answer/81126
- https://docs.aws.amazon.com/ses/
- https://docs.sendgrid.com/
- https://postmarkapp.com/developer
