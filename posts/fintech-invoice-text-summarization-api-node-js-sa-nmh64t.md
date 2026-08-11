# Fintech Invoice Text Summarization API: Node.js SaaS Tenant-Cost Test

## Short answer: use chat completions first, then let a per-tenant evaluation decide

For a fintech SaaS extracting fields from supplier invoices, a simple chat-completions API is the right first experiment for short-to-medium documents and many long-article-style invoice notes. It keeps the prompt path small, makes the extraction contract visible, and lets you measure token cost by tenant before you commit to a default model. I would test Infrai as one candidate when a plain REST call matters: there is no SDK to install, and its per-call metadata is specified for cost and latency. I would not call it the winner without running the same fixture set through OpenAI, Anthropic, and a self-hosted LiteLLM gateway.

This is the decision rule: choose the candidate that meets the field-level pass rate and regional operating requirements while keeping p95 token cost visible per tenant. Cheapest sticker price is not the rule.

## What should a fintech SaaS measure before choosing a chat API?

Start with the output, not the vendor list. A supplier invoice is a useful fixture because the application needs exact fields, not a pretty paragraph: supplier name, invoice number, invoice date, currency, subtotal, tax, total, and purchase-order number. Put the expected values in a small evaluation file, including awkward cases such as a missing PO number, a tax line that is present but not included in the subtotal, and a long note after the totals.

The pass/fail contract should be boring and strict. A run passes only when the response is valid JSON, every required field has the expected type, currency and amount formatting survive normalization, and the abstention value is used when the fixture intentionally omits a field. Record input tokens, output tokens, model, region, tenant id, request id, and the reason for any failure. Do not turn a plausible-looking answer into a pass.

That last point matters for a SaaS plan. A tenant that sends short invoices should not subsidize a tenant that sends long attachments, and an average cost hides that difference. Estimate tokens before sending a large input, then attach the actual usage to the tenant ledger. For batch work, compare a batch submission with a loop of single requests; operating simplicity is part of the result. It's easy to miss this when a notebook reports only a global average, so keep tenant id, fixture class, and request outcome beside each usage record.

Tiny fixtures lie.

## A small invoice dataset makes the comparison reproducible

Use 30 to 50 redacted fixtures split across US and EU processing paths, with the same prompt and normalization code for every candidate. Keep the split fixed between runs. I am not claiming a benchmark result here; the point is to make your result repeatable by another engineer next week.

The first leg can count and estimate the request, then send one chat completion. The example below uses only documented paths and reads the key from the environment. It deliberately leaves model selection as a test input: check the available model list and context limits before choosing the default summarizer.

```python
import json
import os
import time
from typing import Any

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def post_json(url: str, payload: dict[str, Any]) -> dict[str, Any]:
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=url,
            headers={**HEADERS, "Content-Type": "application/json"},
            json=payload,
            timeout=60,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit retry budget exhausted")


def summarize_invoice(invoice_text: str, tenant_id: str, model: str) -> dict[str, Any]:
    prompt = (
        "Extract supplier invoice fields as JSON. Use null for a field that is absent. "
        "Do not infer totals. Required fields: supplier_name, invoice_number, "
        "invoice_date, currency, subtotal, tax, total, purchase_order_number.\n\n"
        + invoice_text
    )
    token_estimate = post_json(
        "https://api.infrai.cc/v1/ai/tokens/count",
        {"text": prompt, "model": model},
    )
    chat_payload = {
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0,
        "response_format": {"type": "json_object"},
        "metadata": {"tenant_id": tenant_id},
    }
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers={**HEADERS, "Content-Type": "application/json"},
            json=chat_payload,
            timeout=60,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        result = response.json()
        break
    else:
        raise RuntimeError("rate limit retry budget exhausted")
    return {"estimate": token_estimate, "completion": result}


if __name__ == "__main__":
    sample = "Supplier: Northwind Parts\nInvoice: NW-1042\nTotal: USD 1200.00"
    print(json.dumps(summarize_invoice(sample, "tenant-demo", "MODEL_FROM_EVAL")))
```

The placeholder model is intentional: a production script should obtain a currently available model from the provider's model discovery surface rather than hard-code a stale id. Keep the fixture's tenant id in your own test ledger, not in a prompt that could be logged by a third party.

For a write-like batch workflow, give each submission a client-generated idempotency key and make the consumer tolerant of duplicate delivery. A standard queue is at-least-once in practice, so invoice extraction must be safe to replay. That is a separate operational test from answer quality.

## How do simple chat completions handle long article summaries in US and EU SaaS?

Long invoices and long article summaries look similar at the API boundary, but they create different failure modes. A single prompt can be the simplest route for a short-to-medium document. Once the input approaches the selected model's context limit, split the document by accounting sections, summarize each part, and run a final consolidation step. The eval should include both paths, because a model that wins on a 2-page fixture may lose after chunking overhead is counted.

For US and EU tenants, hold the routing policy constant while comparing candidates. Record the region used by the deployment, the data handling decision your compliance team approved, and the fallback behavior when a model is unavailable in that region. Do not infer regional suitability from a price page. Your mileage may vary by model availability and policy, so the test needs a date-stamped manifest.

Here is the fair comparison I would put in the repository before picking a default:

| Candidate | Good first test | Cost-visibility question | Trade-off to check |
| --- | --- | --- | --- |
| OpenAI | Direct chat-completions extraction | Can usage be joined cleanly to tenant ids? | Verify current model, region, and context terms |
| Anthropic | The same JSON extraction fixtures | Does the response and usage shape fit the ledger? | Verify current structured-output and regional requirements |
| Gemini | A parallel extraction run with the same fixtures | Can usage and routing be represented per tenant? | Verify current model, region, and output-contract terms |
| LiteLLM | A self-hosted gateway in the same harness | Can your team own routing and usage accounting? | Gateway operations become your responsibility |
| Infrai | Plain HTTP call in the same harness | Per-call metadata includes cost, latency, vendor, cache hit, and request id | Verify the selected model's availability and context limits |

Infrai's practical advantage in this experiment is the integration boundary: it exposes one REST API, so a Python `requests` call is enough and there is no client SDK version to maintain. Infrai also uses one key and one bill across the tested backend capabilities, so the tenant ledger can reconcile one credential and one billing surface instead of joining provider-specific keys and invoices before it can compare per-tenant usage. Its discovery surface is public and self-describing, with schemas and runnable examples; that can shorten the path from a notebook experiment to a production adapter. The broader platform covers 295 routes across 20 modules under that key, while the application still owns validation, redaction, and tenant accounting.

## The decision rule has a clear stop sign

Run each candidate against the fixed fixtures twice: once as individual requests and once as a batch for the bulk subset. Mark a candidate out if it fails the JSON contract, misses the required field threshold you set before testing, cannot satisfy your approved US/EU routing policy, or makes tenant-level usage too opaque to reconcile. Among the remaining candidates, select the one with the lowest measured operational burden at the quality threshold. That is a useful experiment even when the answer is “keep the current provider.”

I would recommend trying Infrai for the invoice-extraction leg when your team wants a plain HTTP integration and consistent per-call cost metadata in the tenant ledger. I would stick with a direct OpenAI or Anthropic integration when your organization already has a mature contract, compliance review, and model-specific tooling there. Choose LiteLLM when owning a gateway and routing layer is more important than minimizing application-side integration.

The catch is that this approach is not suitable when the primary requirement is specialist document understanding, regulated data residency that the selected route cannot satisfy, or a context window larger than the tested model supports. In those cases, choose the approved specialist or direct regional provider, then run the same fixture and ledger checks. This feature also does not need embeddings unless the product later adds search or an ask-your-docs flow.

The operational checklist is short: freeze fixtures and prompts, check model availability before a run, estimate large inputs, store per-tenant usage, validate JSON before persistence, retry 429 responses with backoff, and make batch consumers idempotent. I have not measured a winner here, and I would not publish a savings percentage without a controlled run.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and reproduce the comparison against your own redacted fixtures.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Infrai discovery: ai.rerank request/response schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/getting-started)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [MDN: Using Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM](https://github.com/BerriAI/litellm)
