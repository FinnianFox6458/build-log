# Transactional Email Templates — Create, Preview, and Send Welcome Links with Variables

Short answer: use a reusable welcome-email template with four explicit stages: create, preview, send, and audit. Keep the verification-link token in application code, pass only safe display variables to the template, and retain enough delivery evidence to explain what happened for a particular signup.

This is a good fit for a first transactional-email implementation because copy changes stay out of the deploy cycle. It also makes the compliance question concrete: can the team show which message was rendered, for whom, from which template revision, and what the provider later reported? The message is a welcome email that carries a verification link, not an email OTP flow. There is no hosted email OTP API in this design, so a code-based fallback would need to be built and operated by the application.

## The data flow before the first send

At signup, the application creates a short-lived verification record and a link that points back to its own account service. It then sends four ordinary template variables: the recipient's name, company, login link, and trial dates. The template owns presentation; the application owns authorization, expiry, and token handling.

That separation matters. Marketing can revise a heading without changing token logic, while an engineer can change token policy without asking someone to edit HTML. I keep a template identifier and revision beside the message record, too. A rendered preview is useful evidence, but it is not proof that the final provider accepted the send.

Here is the small API adapter I would put behind the signup worker. It keeps the request boundary explicit, so the same call can be exercised from a notebook and from a scheduled worker. The example previews an existing template; creation and sending remain separate calls in the surrounding workflow.

```python
import json
import os
import time
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


def preview_template(template_id: str, variables: dict[str, str]) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/v1/email/template/preview/{template_id}"
    payload = json.dumps({"variables": variables}).encode("utf-8")

    for attempt in range(4):
        request = Request(
            url,
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": f"preview-{template_id}-{attempt}",
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"preview failed with HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"preview failed with HTTP {error.code}: {detail}") from error
            delay = error.headers.get("Retry-After")
            time.sleep(float(delay) if delay else 2**attempt)
        except URLError as error:
            if attempt == 3:
                raise RuntimeError(f"preview could not reach the API: {error.reason}") from error
            time.sleep(2**attempt)


if __name__ == "__main__":
    print(preview_template("welcome", {
        "name": "Ada",
        "company": "Example Shop",
        "login_link": "https://example.com/verify?t=opaque-token",
        "trial_start": "2026-10-29",
        "trial_end": "2026-11-12",
    }))
```

The escaping is not decoration. A user's name and company are input data, and the link is an HTML attribute. The token itself should be opaque, single-use, and checked by the account service; putting a raw password, a permanent authorization token, or unnecessary personal data into the template makes later review harder.

Do not start with send.

## How should a nodejs transactional email template create and preview flow work?

Preview is a content check. Send is an external side effect. Treating them as the same operation is the easiest way to send a malformed link to real customers.

The workflow is to create a named welcome template, preview it with representative values, and have a person or automated assertion inspect the subject, link target, dates, and unsubscribe or support language required by the product's policy. Only after that check does the signup worker submit the send request. Store a content hash, template identifier, recipient, request identifier, and timestamp in the application's audit record. I initially treated preview as a visual nicety; the useful correction is to treat it as the boundary where unsafe variables and incorrect link targets are rejected before a side effect.

The distinction pays off.

If the provider supports reusable templates, the four operations map cleanly to create, preview, update, and send. A unified REST surface such as Infrai is attractive here when breadth behind one contract matters: email templates and other backend capabilities can be reached through one consistent authentication and request model, so adding a capability is another endpoint rather than another integration. That convenience does not remove the need to preserve application-side audit records or to understand the provider's delivery semantics.

Retries deserve their own decision. A worker can time out after the provider accepted a message. Retrying blindly may produce two welcome emails. Use a client-supplied idempotency key where the sending surface supports one, and persist the association before retrying. On a rate limit, honor the server's retry delay and use exponential backoff. A 4xx response should be recorded with its body, not flattened into a generic “send failed” message.

## Which service fits the evidence requirement?

The right choice depends on where evidence must live and how much delivery infrastructure the team wants to own. Amazon SES is a low-level sending service with strong AWS integration and configuration options; it is a sensible choice when the organization already centralizes identity, logs, and mail-domain controls in AWS. The trade-off is that the application has to assemble more of the template, event, and audit workflow.

SendGrid is a broader email platform with dynamic templates and event-oriented tooling. Its template editor can suit a product and marketing partnership, while its event model needs to be mapped carefully into the application's audit vocabulary. Postmark is focused on transactional mail and separates transactional messaging from broadcast concerns, which can make operational intent clear. It is worth checking its retention and export behavior against the evidence period your compliance team requires rather than assuming “delivered” is a permanent record.

An all-in-one backend API can be the pragmatic fourth option for a small team already integrating several capabilities. The value is consistency and breadth, not a magic compliance certificate. In this workflow, it is a good fit when a single contract reduces integration surface and the team is willing to build a small scheduled job for delivery and open-status retrieval. Events are pull-only here, so do not design around webhook callbacks or promise signup-state updates in real time.

| Option | Useful strength | Boundary to verify |
| --- | --- | --- |
| Amazon SES | AWS-native sending and domain controls | More workflow assembly belongs to the application |
| SendGrid | Dynamic templates and email event tooling | Map event history and retention to your evidence policy |
| Postmark | Transactional-email focus | Confirm retention and export requirements |
| Unified backend API | One contract across multiple backend capabilities | Poll status; confirm the needed vendor and regional readiness |

The compliance evidence decision should be made with a sample record, not a feature checklist. Send one message in a non-production environment, render the exact variables, capture the provider response, run the status poll, and ask another engineer to reconstruct the timeline from stored records. That test exposes missing identifiers faster than a polished demo does.

## What should the worker record and poll?

The scheduler is intentionally boring. Every few minutes, fetch the pending message records, query the email event list for the provider identifiers, and write an append-only status observation. Keep the original send request and later observations distinct; overwriting a status loses the chronology that an audit reviewer usually wants.

I would record recipient, template revision, variable names rather than sensitive values, content hash, send request ID, provider message ID, timestamps, status, and the last error body. Redact links if they contain bearer-like tokens. A failed poll should remain visible as an operational gap, not silently turn a message into “unknown.”

This design does not claim more than the channel provides. There is no SMTP relay, no hosted email OTP interface, and no webhook event push in the stated surface. Email scheduled sends also have no cancellation interface, so a delayed welcome message should be prevented by application state before submission, not “canceled” after the fact. Domestic provider readiness must also be checked separately; a pending vendor is not evidence of domestic compliance.

## A practical go-live rule

Ship when a template revision can be previewed with safe fixture data, a verification link can be tested as single-use and expired, duplicate-send behavior is defined, and an operator can reconstruct one message from signup through the latest polled event. Keep the cron interval and retention period as explicit configuration, then exercise the path with test recipients before enabling production traffic.

The best implementation is rarely the one with the longest provider feature list. It is the one whose evidence still makes sense six months later, after a copy edit, a retry, a delayed event, and a support question from a new account holder.

## References

- Amazon SES Developer Guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SendGrid Dynamic Templates: https://docs.sendgrid.com/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- SendGrid Event Webhook documentation: https://docs.sendgrid.com/for-developers/tracking-events/event
- Postmark Templates API: https://postmarkapp.com/developer/api/templates-api
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
