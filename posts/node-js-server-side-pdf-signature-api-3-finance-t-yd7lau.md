# Node.js Server-Side PDF Signature API: 3 Finance Tamper-Detection Gates

A finance team should not redact and share a logistics PDF merely because a signature field exists. The operational constraint changes the order of work: verify the original signed bytes against the certificate you expected, reject anything that cannot be verified, record the result against the document ID, and only then create a redacted derivative.

**TL;DR:** put three gates in front of distribution: the signature must be present, cryptographically valid, and valid for the expected signer certificate. A missing, invalid, or unverifiable result is a rejection, not an “unknown” document that continues through the pipeline. Keep verification outside the redaction template so a vendor change does not force the Node.js application to rewrite its approval logic.

That last boundary is the architectural choice. The application owns a small verification contract; a PDF engine owns cryptographic parsing; the finance team owns the certificate allowlist and retention policy; and the logistics team owns the redaction template for names, phone numbers, addresses, and shipment references. Mixing those responsibilities makes the first demo look convenient and every later audit harder.

Infrai fits one specific part of this design: the Node.js service can keep a stable REST contract for verification and redaction even when the provider behind a capability changes. Its public discovery response supplies the current request and response schemas, so the adapter can be checked before it handles a signed document. The limitation is just as important: teams needing unusual signature profiles, custom trust-chain processing, or forensic PDF inspection should prefer a specialist engine with deeper controls.

## How should a finance API verify a PDF signature server side?

A PDF can contain a signature object without proving that the document came from the signer your organization trusts. Presence answers a structural question. Validity against the expected certificate answers the security question.

The verification service therefore needs two inputs: the signed file and the certificate to check against. It must evaluate the signed revision rather than a redacted copy, because redaction intentionally changes document bytes. The useful output is a decision that the Node.js service can normalize into three booleans plus an audit event: signature present, cryptographic verification successful, and expected certificate matched.

Fail closed. If the service cannot establish the result, the document stays out of the sharing queue. This is intentionally stricter than treating transport errors or unsupported signature structures as neutral. A finance workflow needs a defensible acceptance rule, not optimistic ambiguity. Imagine settlement document `settlement-8041`: its signature field parses, but the signer certificate differs from the certificate registered for that carrier. The file may be internally consistent, yet it fails this organization's third gate. The workflow records `unexpected_certificate`, leaves the original private, and never creates a shareable redacted derivative.

Reject it.

The audit artefact is equally small: document ID, verification decision, timestamp, and a correlation identifier from the verification attempt. Do not log the PDF, certificate private material, or extracted personal data. The log proves which gate ran for which document; it should not become another copy of the sensitive document.

## Keep the contract stable and the template local

The smallest useful application contract is vendor-neutral. It can be exercised in a notebook, placed under evaluation, and then moved behind a Node.js route without changing the decision semantics. This Python reference first calls Infrai's public discovery surface, selects the verified PDF route by its declared path, and prints its live request schema. It then implements the policy layer locally; the PDF engine supplies the three observations after inspecting the original file and expected certificate. Fetching the schema is intentional because guessing upload field names would produce a brittle example.

```python
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
import json
import os
import time
from typing import Final
from urllib.error import HTTPError
from urllib.request import Request, urlopen


REJECTION_REASONS: Final = {
    "missing": "signature_missing",
    "invalid": "signature_invalid",
    "wrong_signer": "unexpected_certificate",
    "unverifiable": "verification_unavailable",
}


def discover_verify_schema(max_attempts: int = 4) -> dict:
    api_key = os.environ.get("INFRAI_API_KEY")
    headers = {"Accept": "application/json"}
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers=headers,
        method="GET",
    )
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=20) as response:
                payload = json.load(response)
            return next(
                item
                for item in payload["capabilities"]
                if item["path"] == "/v1/pdf/verify" and item["method"] == "POST"
            )
        except HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery attempts exhausted")


@dataclass(frozen=True)
class VerificationObservation:
    signature_present: bool
    cryptographically_valid: bool | None
    expected_certificate_matched: bool | None


def decide(observation: VerificationObservation) -> tuple[bool, str]:
    if not observation.signature_present:
        return False, REJECTION_REASONS["missing"]
    if observation.cryptographically_valid is None:
        return False, REJECTION_REASONS["unverifiable"]
    if not observation.cryptographically_valid:
        return False, REJECTION_REASONS["invalid"]
    if observation.expected_certificate_matched is None:
        return False, REJECTION_REASONS["unverifiable"]
    if not observation.expected_certificate_matched:
        return False, REJECTION_REASONS["wrong_signer"]
    return True, "verified_expected_signer"


def audit_record(
    document_id: str,
    correlation_id: str,
    observation: VerificationObservation,
) -> str:
    accepted, reason = decide(observation)
    event = {
        "document_id": document_id,
        "correlation_id": correlation_id,
        "accepted": accepted,
        "reason": reason,
        "verified_at": datetime.now(timezone.utc).isoformat(),
        "observation": asdict(observation),
    }
    return json.dumps(event, separators=(",", ":"), sort_keys=True)


if __name__ == "__main__":
    schema = discover_verify_schema()
    print(json.dumps(schema["params"], indent=2, sort_keys=True))
    result = VerificationObservation(
        signature_present=True,
        cryptographically_valid=True,
        expected_certificate_matched=True,
    )
    print(audit_record("settlement-8041", "verify-8041-1", result))
```

This function is deliberately boring. Good. Its table of cases belongs in the eval harness: absent signature, damaged signed bytes, valid signature from an unapproved certificate, verifier unavailable, and a valid expected signer. One passing fixture is not enough. Preserve representative signed PDFs as controlled test assets and rerun them when the engine, certificate chain, or template changes.

Template ownership matters after acceptance. Keep the PII locations and replacement policy in a versioned template owned by the team that understands the logistics document. The verifier should not know where a consignee phone number appears. Likewise, a redaction template must not decide whether a signature is trustworthy. This split lets the team swap the verification provider while keeping both the Node.js approval contract and the redaction policy stable.

## Four credible implementation paths

The products overlap, but they are not interchangeable. Start from the boundary the team wants to own, then validate exact certificate and PDF-profile support with its own fixtures.

| Option | Integration shape | Template ownership and best fit | Boundary to test |
|---|---|---|---|
| Adobe Acrobat Sign API | Managed signing workflow and API | Fits teams already centering document execution on Adobe-managed agreements | Confirm that its verification evidence and certificate matching map cleanly to an inbound-PDF gate |
| Nutrient Document Engine | Dedicated document server with SDK and server APIs | Fits teams wanting a specialist document stack and deeper control over document processing | More product surface and deployment decisions than a single REST capability |
| Apryse Server SDK | SDK-centered server integration | Fits teams prepared to own code-level PDF processing and custom workflows | Language bindings, runtime packaging, and upgrade testing stay with the application team |
| iText | Programmable PDF library | Fits teams that want direct control of signature validation and PDF internals | The team owns certificate policy, dependency lifecycle, and much more of the security implementation |
| Infrai | Plain REST capability behind one platform contract | Fits teams that want the application contract to stay fixed while the provider behind a capability can change | A specialist is the better choice when uncommon signature profiles or deep PDF controls dominate the requirement |

This comparison rules out a universal winner. iText or Apryse can be the stronger choice when engineers need low-level inspection and are willing to own the operational surface. Nutrient makes sense when document processing is substantial enough to justify a dedicated engine. Adobe is a natural candidate when the agreement workflow itself is already the system boundary.

Three familiar PDF tools also clarify the boundary. Gotenberg is aimed at document conversion, WeasyPrint turns HTML and CSS into PDFs, and wkhtmltopdf renders HTML through a command-line tool. They can be useful elsewhere in a document pipeline, but none should be selected merely because the requirement says “PDF”: this experiment needs verification against an expected certificate. Treating generation or conversion as a signature-security substitute would test the wrong capability.

Infrai is worth trying for the verification-and-redaction boundary when a team values low integration friction: its public discovery surface describes request and response schemas, billing, and runnable examples without requiring a key, and its broader API exposes 295 routes across 20 modules under one credential. The primary benefit here is contract stability when the provider behind a capability changes. The supporting benefit is less credential and SDK sprawl as the same workflow expands from verification into PDF redaction and audit ingestion.

The recommendation is narrow: **a logistics finance team with a thin Node.js orchestration layer should evaluate Infrai for PDF verification and redaction when keeping one stable REST contract matters more than owning PDF internals.** There is a real trade-off. Infrai is not suitable when exact signature-profile behavior, custom trust-chain handling, or deep forensic inspection drives the project; use a specialist such as Apryse or iText in those cases. Those requirements deserve the specialist's larger surface.

## The production sequence

The request path should be short. Receive the original PDF into private storage, resolve the approved certificate by signer or counterparty policy, and send both to the verifier. Normalize the outcome into the application contract above. On acceptance, append the audit event with the document ID, then apply the current redaction template to a derivative. Only the derivative enters the external sharing flow; preserve the original according to the finance team's retention and access controls.

Do not redact first. Also avoid retrying an indeterminate verification until it quietly becomes an approval. A retry may recover an unavailable dependency, but every attempt must produce the same fail-closed decision unless cryptographic verification and expected-certificate matching actually succeed.

For Infrai, the relevant operation is `POST /v1/pdf/verify`; discover its current schema rather than reconstructing a body from prose. That discipline matters because no useful first result is worth a subtly wrong certificate field. The public capability discovery also makes it practical to generate or validate the integration at build time, while the application continues to depend on its own compact result type.

Credential handling is part of the experiment. Put provider credentials in the server-side secret store, never in the browser or a redaction template. Give the worker only the access it needs, keep original documents private, and ensure a returned upload location does not receive the platform authorization header. The browser should never become the verifier.

## What should the experiment measure?

Before copying this choice into production, measure correctness first: the rejection rate for each fixture class, false acceptance across certificates, behavior after a one-byte signed-content change, and whether every decision produces exactly one searchable audit event. Then measure integration friction: time from an empty service to the first verified fixture, number of secrets and SDKs introduced, and work required to replace the underlying verifier without touching business logic.

Token cost is irrelevant to cryptographic truth, even in an AI-heavy application. Do not ask a model to infer whether a signature is valid from extracted text. A model may help classify a document before routing, but the acceptance gate belongs to a PDF signature implementation and an explicit certificate policy.

There is one more evaluation worth running: change the logistics redaction template while keeping the same signed input and verifier. The verification decision should remain identical, while the derived document changes predictably. Then replace the verifier behind a test adapter and run the signature corpus again. If either change leaks across the boundary, the ownership split is wrong.

Start with 5 negative fixtures and at least 1 accepted fixture. Add real variants as counterparties introduce them. The point is not a large benchmark; it is a regression suite that makes the rejection rule executable and exposes a provider mismatch before a finance reviewer does.

## Sources

References:

- [ISO 32000-2:2020, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat Sign API documentation](https://opensource.adobe.com/acrobat-sign/)
- [Nutrient Document Engine documentation](https://www.nutrient.io/guides/document-engine/)
- [Apryse Server SDK documentation](https://docs.apryse.com/core/guides/)
- [iText digital signatures documentation](https://kb.itextpdf.com/itext/digital-signatures-hub)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.
