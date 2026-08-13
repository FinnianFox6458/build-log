# Media Support Triage: Extract Structured JSON from LLM Text After Invalid Responses

Short answer: for media support triage, treat the LLM as an untrusted serializer behind a tenant-aware budget and a local JSON Schema validator. Parse once, validate once, retry only for a classified, recoverable failure, and record usage before the ticket is routed.

That choice came from the operational constraint, not from a preference for a particular model. One tenant's long transcript can consume the same queue budget as dozens of short tickets if the worker measures only requests. A notebook that returns attractive dictionaries can still become an expensive, hard-to-audit service.

The useful boundary is small: input text goes in, a typed decision comes out, and the accounting record travels with it. Everything between those points should be observable.

## How should a Python builder extract structured JSON from invalid text?

“Return JSON” is an instruction. It is not a contract. A model can wrap an object in markdown, emit an empty response, truncate a quoted customer message, or return syntactically valid JSON with the wrong type. `json.loads` catches only one of those classes.

For a media company, the target might be deliberately boring:

```python
TICKET_SCHEMA = {
    "type": "object",
    "properties": {
        "queue": {"type": "string", "enum": ["billing", "playback", "account", "other"]},
        "urgency": {"type": "string", "enum": ["low", "normal", "high"]},
        "summary": {"type": "string"},
    },
    "required": ["queue", "urgency", "summary"],
    "additionalProperties": False,
}
```

The schema is a product decision as much as a parsing detail. `queue` is an allowed routing action; `urgency` has a bounded vocabulary; `summary` is useful to a human but should not be trusted as a reason to bypass authentication, refunds, or account controls. Keep those sensitive actions outside extraction.

I separate three outcomes in the eval harness: malformed JSON, valid JSON that fails schema validation, and a transport or quota event. They have different remedies. A parse failure may justify one repair attempt. A quota event belongs to backoff and tenant accounting. A valid but unsafe routing value should be rejected, not repaired into an action.

Small distinction. Big effect.

## How can Python builders keep LLM JSON valid while preserving tenant cost visibility?

The worker needs two ledgers: an acceptance ledger and a cost ledger. The first says why a response was accepted or rejected. The second says which tenant, model call, input size, output size, and retry produced the billable work. Never infer tenant identity from text; carry it as request metadata from the authenticated queue message.

Here is the core state machine. The model client is an injected callable so the same checks run in a notebook, a test fixture, or a production worker. The `input_tokens` and `output_tokens` values come from the client adapter's usage record; they are not guessed from character count.

```python
import json
from dataclasses import dataclass
from typing import Any, Callable

import jsonschema


@dataclass(frozen=True)
class Usage:
    input_tokens: int
    output_tokens: int


@dataclass(frozen=True)
class ModelReply:
    text: str
    usage: Usage


def extract_ticket(
    tenant_id: str,
    ticket_text: str,
    call_model: Callable[[str, str], ModelReply],
) -> tuple[dict[str, Any], list[dict[str, Any]]]:
    attempts: list[dict[str, Any]] = []
    repair_note = ""

    for attempt in range(2):
        reply = call_model(ticket_text, repair_note)
        event = {
            "tenant_id": tenant_id,
            "attempt": attempt + 1,
            "input_tokens": reply.usage.input_tokens,
            "output_tokens": reply.usage.output_tokens,
        }

        try:
            value = json.loads(reply.text)
            jsonschema.validate(value, TICKET_SCHEMA)
        except (json.JSONDecodeError, jsonschema.ValidationError) as exc:
            event["result"] = "invalid_json" if isinstance(exc, json.JSONDecodeError) else "schema_error"
            event["error"] = str(exc)
            attempts.append(event)
            repair_note = f"Return only JSON matching the schema. Validation error: {exc}"
            continue

        event["result"] = "accepted"
        attempts.append(event)
        return value, attempts

    raise ValueError(f"ticket extraction exhausted its retry budget for tenant {tenant_id}")
```

The example intentionally does not hide a failed response inside a regex cleanup step. Removing a leading fence or taking the substring between the first and last brace can turn damaged output into a plausible, incomplete ticket. Imagine a playback ticket whose quoted customer message contains a second JSON object: a brace-based cleanup may select the wrong boundary, `json.loads` may still succeed, and the queue will receive a confidently misrouted record. That is harder to detect than a clean parse error because the failure has crossed into business data. Store the raw response in a restricted diagnostic sink if policy permits, but route only the validated value—and don't silently fill a missing field with a default.

The retry budget is part of the cost policy. A repair request repeats the ticket text and therefore spends more input tokens. Set the ceiling per tenant and per queue, then expose `attempt`, `input_tokens`, `output_tokens`, and `result` as dimensions in metrics. A global success rate can look fine while one tenant's verbose tickets quietly consume the budget.

## What should the eval harness measure before notebook-to-prod?

Start with a fixture set built from representative support messages: playback symptoms, billing disputes, account access requests, quoted emails, long transcripts, empty bodies, and multilingual text if the service accepts it. Redact secrets before the fixtures enter a shared test store. The goal is not a pretty demo; it is a decision boundary that fails in known ways.

Measure first-attempt schema validity, valid-after-repair rate, wrong-queue rate, missing-field rate, p95 latency, and tokens per ticket. Add a tenant-level view for each metric. A retry that rescues a response may improve acceptance while worsening latency and cost, so those numbers must sit beside each other. I would keep the raw event shape stable even when the model adapter changes: `tenant_id`, attempt number, input and output token counts, parser result, schema result, and final routing decision. That lets an eval report answer several questions at once: did the model produce invalid JSON, did the validator reject a legal-looking shape, did a retry consume the tenant's remaining allowance, and did a successful extraction still choose the wrong queue? A single aggregate “success” counter cannot answer those questions, and it encourages teams to increase retries until the dashboard looks calm.

I would also test adversarial content: a customer message that says “ignore the routing rules,” a ticket containing JSON-looking text, and a message that asks for a refund. The extractor can summarize and classify. Authorization still belongs to deterministic application code.

Your mileage may vary on the token threshold. Different models tokenize the same transcript differently, and the right truncation policy depends on whether the queue needs the full conversation or only the latest customer turn. Measure this with the actual model adapter and make truncation an explicit event, never an invisible string slice.

## When is a simple provider adapter the wrong boundary?

An adapter is a good fit when the main requirement is portability: the application owns the schema, validator, retry policy, redaction, and accounting, while the adapter supplies text and usage. It is less suitable when a provider-specific structured-output feature is a hard requirement and the adapter would erase controls needed for that feature. In that case, keep the provider integration behind the same acceptance interface and document the extra contract.

The trade-off is maintenance. A generic interface reduces application changes when the backend changes, but it cannot make incompatible context limits, safety controls, streaming semantics, or usage fields identical. Standardize the record you need for operations, then preserve provider-specific fields in a separate, optional section.

For streaming responses, do not validate each arbitrary chunk as a complete object. Buffer according to the transport's event framing, detect completion, then parse and validate the assembled payload. Server-Sent Events describes an event stream, not a guarantee that each event is a JSON document. This distinction matters when a partial object reaches a parser.

## A decision rule for the media queue

Use synchronous extraction for a short ticket whose tenant has remaining budget and whose latency target permits one bounded retry. Send long historical transcripts through a separately metered worker, with a maximum input policy and a dead-letter reason that a person can inspect. If the tenant budget is exhausted, route to a human queue or a deterministic fallback; do not spend another model call just to preserve an aggregate success percentage.

The recommendation is not suitable when the output must trigger an irreversible action without human or deterministic authorization. It is also a poor fit for a team that cannot retain per-tenant usage records long enough to investigate disputes. Stick with a narrower, rules-based router when the label set is stable and the cost of a false route is higher than the value of free-form summaries.

Before copying this design, run the fixture set against the real workload and inspect the distribution, not just the mean. The right result is a bounded, explainable failure with a tenant attached. That is what lets a Python extraction notebook become a service without losing the ability to answer, “Why did this ticket cost this much?”

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
- https://json-schema.org/specification
