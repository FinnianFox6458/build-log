# MVP SaaS Structured Logging Backend: Reconstructing Silent Scheduled Imports by Request ID

A scheduled import can finish without an exception and still produce zero results. That operational constraint changes the logging-backend decision for an MVP SaaS app: test whether an operator can reconstruct the last successful result, the next scheduled attempt, and the missing output from searchable structured events. Short answer: choose a hosted backend only after a timed reconstruction exercise proves that exact-match run and request IDs remain searchable across delayed delivery and retention boundaries. A dashboard showing that a worker is alive does not explain why a developer-tools catalog stopped updating.

## What does a silent import look like in the logs?

The tempting first pass is to forward application logs and alert on errors. It misses a run that exits cleanly after reading an empty page, a scheduler that never dispatches the run, and a result writer that acknowledges a batch while recording no accepted results. These are different incidents. Search needs to join scheduler intent, worker execution, and persisted outcome without guessing a message substring.

No exception is required.

For a concrete evaluation fixture, imagine a daily import with run ID `run-042`, scheduled for `09:00`, linked to request ID `req-8c1`. Those values are illustrative test inputs, not measured production data. Emit separate events for `import.scheduled`, `import.started`, and `import.completed`; include `results_written` on completion. A missing completion differs from a completion with `results_written: 0`. Keep `run_id` stable across retries, give each attempt its own `attempt_id`, and record each outcome. Otherwise duplicate dispatches merge into one misleading story.

Search by `run_id` first, then pivot to `request_id` for the downstream write and to an authorized tenant identifier when investigating affected accounts. User ID search matters for interactive requests, but a scheduled job may have no human user. Making `user_id` mandatory for background work encourages invented values. Never use raw access tokens or imported content as correlation keys.

The join key must survive retries.

## How should an MVP SaaS app test a structured logging backend?

The producer and the storage choice are separate. Pino and Winston are JavaScript logging libraries, not substitutes for a searchable backend; for a Python import worker, the standard logging module can produce the same explicit event contract. A logger can serialize useful fields. It cannot guarantee that a backend indexes them, preserves their types, or makes them available before an alert is evaluated. The issue is not which library wins a popularity contest.

In a notebook-to-prod workflow, I would start with one synthetic run and an evaluation sheet: can another engineer find the last successful run, distinguish an absent start from a zero-result finish, and locate the retry without knowing the wording of the messages? This is a proposed experiment, not a reported benchmark. The following Python example shows the producer-side contract. It writes one JSON object per line to standard output; the scheduler and persistence layer must emit their own events at the points they actually control. A completion event belongs after the result write succeeds, not after a fetch returns.

That ordering is the test.

```python
import json
import logging
from datetime import datetime, timezone


class JsonFormatter(logging.Formatter):
    def format(self, record):
        event = {
            "time": datetime.fromtimestamp(record.created, timezone.utc).isoformat(),
            "severity": record.levelname,
            "event": record.getMessage(),
        }
        for field in ("run_id", "attempt_id", "request_id", "results_written"):
            if hasattr(record, field):
                event[field] = getattr(record, field)
        return json.dumps(event, separators=(",", ":"))


logger = logging.getLogger("imports")
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info(
    "import.completed",
    extra={
        "run_id": "run-042",
        "attempt_id": "attempt-1",
        "request_id": "req-8c1",
        "results_written": 0,
    },
)
```

This code does not implement a delivery guarantee. A process can terminate before buffered output is collected, and a collector can lag behind the application. Test both conditions. Compare the scheduler's expected-run record against durable completion records or a heartbeat derived from persisted results; logs explain the incident, while the independent expected-versus-observed check detects silence. Zero results can also be legitimate for a source. Don't page on that number alone.

This distinction matters most during a rollout: if the collector receives `import.completed` late, searching the run by ingestion time alone can make the job look absent, while searching only event time can hide the delivery lag. Preserve both timestamps in the evaluation record and ask the investigator which one the search UI uses to filter the result. For an import that crosses a retry boundary, compare the attempt IDs before attributing a zero-result completion to the latest attempt. That extra query may change which component the on-call engineer investigates first.

## Which search capabilities survive the fixture?

Give each hosted candidate the same five cases: a successful run, a missing dispatch, a zero-result completion, a retry, and delayed delivery. Evaluate exact-ID queries inside a bounded time range. Ask whether structured numeric fields remain numeric, whether late events become discoverable, whether tenant permissions restrict access, and whether exported events retain original IDs and timestamps. Check retention against the longest realistic investigation delay. Fast search is useless after its evidence expires.

| Fixture | Evidence the investigator needs |
| --- | --- |
| No dispatch | Expected schedule exists; start event is absent |
| Empty finish | Completion exists with numeric `results_written: 0` |
| Retry | One run ID links distinct attempt IDs and outcomes |
| Late delivery | Original event time remains separate from ingestion time |

Measure reconstruction time, manual query pivots, ingestion-to-search lag, and the fraction of expected events that arrive. Also measure bytes ingested and indexed under representative payload sizes. AI features make indiscriminate prompt logging especially costly and risky; log token counts, model operation identifiers, and evaluation-run IDs when they serve an investigation, while keeping prompt text out of routine event streams. Use a separate access-controlled eval dataset when actual examples require content inspection.

Richer indexed fields improve search but increase storage and disclosure exposure. Start with a compact schema and explicit retention. If a backend only offers free-text search or silently drops nested fields, the experiment exposes the limitation before migration is painful. If two candidates pass, weigh export, permissions, and operational ownership. Low cost matters to an MVP, but a headline ingestion price cannot settle the decision.

Keep the original event.

## What should the alert actually say?

Alert on expected output missing past a declared grace interval, with the source, scheduled window, last persisted result time, and a linkable run identifier. Derive the interval from measured source variability and collection delay, not an arbitrary constant copied from another job. A missing scheduler record calls for a different owner than a completed run with zero accepted results. Keep those states separate in the alert payload.

Before adoption, replay the five fixtures through the real collection path and ask an engineer who did not write the queries to explain each outcome. Record measured detection lag and reconstruction time, then repeat after changing retention or adding another source. The criterion is practical: the team can prove where results stopped with bounded search effort, while the alert still works when no error log exists.

## References

- Python logging documentation: https://docs.python.org/3/library/logging.html
- OpenTelemetry logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html

## Sources

- https://docs.python.org/3/library/logging.html
- https://opentelemetry.io/docs/specs/otel/logs/data-model/
- https://www.w3.org/TR/trace-context/
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
