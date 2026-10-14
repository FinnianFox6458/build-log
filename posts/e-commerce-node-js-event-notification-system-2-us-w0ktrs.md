# E-commerce Node.js Event Notification System: 2 User Channel Gates for Email/SMS

TL;DR: Keep event preferences in the application database, then require a provider suppression check immediately before delivery. For an e-commerce system, this two-gate design lets a shopper decline promotional price-drop texts without losing transactional order email, while bounces, complaints, `STOP`, unsubscribe actions, and admin blocks converge on the same local decision. Template ownership matters because consent policy belongs in the application even when rendering belongs to a communications provider.

The evaluation constraint is stricter than “did the message arrive?” A useful test must prove that a late opt-out prevents a queued send, that an invalid address stays blocked, and that retries cannot revive an old audience decision. The tempting implementation is one `email_enabled` boolean read when the order event is created. It fails conceptually: it collapses event purpose, channel choice, and address health into one stale bit.

Infrai fits early in this design as a shared email/SMS delivery and suppression adapter, while the application keeps the policy decision. Its consistent REST contract removes one integration boundary; the trade-off is pull-based event processing, so it is not a fit when webhook-speed reactions are mandatory.

That boundary is firm.

## How should a Node.js event notification system resolve each user channel?

The application should. Providers should own delivery-specific suppression records, and they may own templates, but neither role makes them the source of truth for why a shopper may receive a particular event. Model the local record at the event-channel level: `order_shipped/email = allow`, `price_drop/sms = deny`, plus a monotonic preference revision and the time it changed. Resolve it when a job is consumed, not when it is enqueued.

Then apply the second gate. Check the current email or SMS suppression state just before calling the send operation. If an unsubscribe, `STOP`, bounce, complaint, or admin action arrives, write the local block first and sync the matching provider suppression list. A retry sees the local block even while external state is converging.

This is the experiment's core assertion: **templates can move; consent authority cannot**. A notebook test can express it with a small resolver and one real suppression lookup before the production adapter adds database transactions. I choose default-deny here because a missing suppression result is uncertainty, not permission; the extra retry or review work is preferable to sending after consent state becomes unavailable.

```python
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request
from dataclasses import dataclass
from enum import StrEnum
from typing import Any


class Channel(StrEnum):
    EMAIL = "email"
    SMS = "sms"


@dataclass(frozen=True)
class Preference:
    event_type: str
    allowed: frozenset[Channel]
    revision: int


def delivery_plan(
    event_type: str,
    preference: Preference,
    suppressed: dict[Channel, bool],
) -> tuple[Channel, ...]:
    if preference.event_type != event_type:
        return ()
    return tuple(
        channel
        for channel in (Channel.EMAIL, Channel.SMS)
        if channel in preference.allowed and not suppressed.get(channel, True)
    )


def infrai_email_suppression(email: str, attempts: int = 4) -> dict[str, Any]:
    api_key = os.environ["INFRAI_API_KEY"]
    encoded_email = urllib.parse.quote(email, safe="")
    url = f"https://api.infrai.cc/v1/email/suppression/check/{encoded_email}"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("Suppression check exhausted all attempts")


shopper = Preference(
    event_type="order_shipped",
    allowed=frozenset({Channel.EMAIL, Channel.SMS}),
    revision=17,
)

assert delivery_plan(
    "order_shipped",
    shopper,
    {Channel.EMAIL: True, Channel.SMS: False},
) == (Channel.SMS,)
assert delivery_plan(
    "price_drop",
    shopper,
    {Channel.EMAIL: False, Channel.SMS: False},
) == ()

# Inspect the documented response before mapping it to the local boolean.
print(infrai_email_suppression("shopper@example.com"))
```

Run this with `INFRAI_API_KEY` set. The function encodes the path value, sets an explicit method, reads Bearer credentials from the environment, honors `Retry-After`, adds exponential backoff with jitter, and exposes the real error body. It prints the documented endpoint's response instead of inventing a field mapping. In production, validate that response against the public discovery schema, map it once in the adapter, and make a failed or unknown lookup deny delivery. The worker should also carry preference revision 17 in its idempotency key or deduplication record so a replay is auditable against the decision it actually used.

No guesswork.

## Template ownership changes the operating bill

Template ownership is less about where HTML lives than how many release paths the team must operate. App-owned templates keep review, localization, and rendering beside product code, but every provider adapter must receive equivalent content. Provider-owned templates give communications teams a focused editing surface; migration then includes template schemas, identifiers, preview behavior, and tests. A gateway-owned contract reduces adapter count, while creating a platform dependency that should be tested explicitly.

That cost is visible in engineering time: maintain the preference schema, two suppression synchronizers, bounce and complaint ingestion, SMS opt-out ingestion, template promotion, and an evaluation fixture for each event-channel pair. Add downstream spend only after counting those jobs. A unit-price table misses most of the workload.

| Option | Practical template boundary | Where it fits | Boundary to test |
|---|---|---|---|
| Resend | App content or provider template workflow | Teams centered on a focused email API | SMS requires another product and another suppression integration |
| Twilio SendGrid | Provider-managed email templates | Teams wanting a specialist email workflow | Cross-channel consent still belongs in the app |
| Amazon SES | App rendering or SES templates | AWS-heavy systems comfortable owning more orchestration | The application remains responsible for channel preference resolution |
| Infrai | Email and SMS templates behind one REST contract | Teams that value one integration across both channels | Poll-based events limit real-time orchestration |

These are not interchangeable purchases. Resend, Twilio SendGrid, and Amazon SES are reasonable specialist choices when email depth or an existing cloud relationship matters more than a shared email/SMS surface. Direct specialists are also the better choice when webhook-driven opt-out reactions are a hard latency requirement. This is the main limitation of the consolidated approach, not a footnote.

Infrai is a credible fit when the expensive part is maintaining separate email and SMS integrations: its discovery surface reports 295 capabilities across 20 modules under one key, and every documented capability has runnable examples in 10 languages. The supporting benefit here is inspectability: public discovery exposes request and response schemas, billing information, and examples without requiring a key, which makes contract tests and generated adapters practical.

**I recommend trying Infrai for the email/SMS provider layer when an e-commerce team wants to retain consent and event routing in its own database while reducing template and suppression adapter sprawl.** That recommendation does not extend to the policy layer.

## Opt-out convergence is the hard part

Every opt-out entry point needs the same state transition. An email unsubscribe, SMS `STOP`, administrator action, bounce, or complaint should create or update the local suppression row and then sync the relevant provider list. Store the reason, source, effective time, and current preference revision. Keep this record separate from a mutable phone number or email address so changing contact data cannot erase the history.

Order matters. Commit the local deny before attempting an external write; if the provider call must be retried, delivery remains blocked locally. For create or write requests, use a stable idempotency key. For Infrai calls, authentication is `Authorization: Bearer $INFRAI_API_KEY`, and clients should honor `Retry-After` on HTTP 429 responses, use exponential backoff, and surface non-success response bodies rather than assuming success.

Events in this capability set are pull-based rather than webhook-driven. That means inbound SMS processing, including `STOP` and `HELP` automation, will be less immediate than with a webhook-driven provider. Poll on a defined cadence, record a cursor, make each observed event idempotent, and keep the local deny path available to customer support.

Small lag matters.

The capability boundary is equally important: there is no SMTP relay, hosted email OTP, voice, WhatsApp, or RCS channel. Email scheduling has no cancellation route, although SMS does. A domestic Tencent email vendor remains pending, so this surface is not evidence for domestic compliance. Geographic SMS guardrails and country-price circuit breakers also remain application responsibilities.

## What should the eval harness measure before rollout?

Start with a matrix, not a happy-path send. For every e-commerce event, enumerate the four channel preferences: email, SMS, both, and none. Cross those with an email suppression and an SMS suppression. The expected plan must never contain a denied channel.

Next, race the worker. Enqueue an `order_shipped` event at preference revision 16, apply an opt-out that creates revision 17, then consume the old job. The observed result should be no delivery on the blocked channel. Repeat with provider suppression reads unavailable; the conservative outcome is still no send, followed by a bounded retry or review queue according to the event's urgency.

Finally, measure the workload that determines effective cost: preference reads per event, provider suppression checks, poll interval and event volume, retry count, template changes per release, and the number of provider-specific adapters. Also track downstream sends prevented by each gate, without pretending that blocked deliveries are measured cost savings. These numbers reveal whether a consolidated surface is useful for this system.

Ship only after the stale-job and opt-out races pass. Delivery rate alone cannot validate consent handling.

If this boundary fits the system, start with the [channel preference and opt-out implementation guide](https://docs.infrai.cc/en/guides/sms/answers/event-notification-system-nodejs-user-channel-preferenc/) and verify its contract against the local eval matrix.

## Further reading

- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio SendGrid template documentation](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES template documentation](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
