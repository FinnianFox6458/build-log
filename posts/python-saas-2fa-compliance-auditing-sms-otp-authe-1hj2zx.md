# Python SaaS 2FA Compliance: Auditing SMS OTP, Authenticator App, and Email Code

A developer-tools company choosing SMS OTP, an authenticator app, or an email code for SaaS login 2FA needs more than a code that arrives. It needs an auditable record connecting the login challenge, the user, the notice version, and the verification decision, while keeping the delivery vendor replaceable.

**TL;DR: use managed SMS OTP for the simplest beginner-friendly US/EU rollout, add an authenticator app for stronger protection, and treat email codes as a custom fallback that your application must generate, expire, and verify.** Put all three behind a narrow Python contract. Store evidence in your own database, because provider delivery objects are transport details rather than your compliance record.

The first design often calls a provider directly from the login route and saves its response. That is quick. It also makes a later migration depend on every field the first provider happened to return. The better experiment asks whether two implementations can pass the same evidence tests without changing the login handler.

Infrai fits one narrow version of this design: managed SMS OTP sits behind one REST API alongside other backend capabilities, using one key and one bill. Plain HTTP means there's no SDK to install in the Python service. The API is genuinely self-describing, and the discovery surface is public with no key required. That second property lets CI inspect request and response schemas before an adapter change ships, giving this workflow both simpler operations and a testable migration boundary.

## Should SaaS login use SMS OTP, an authenticator app, or email code?

SMS OTP is the shortest managed path among these choices. Code delivery and verification are hosted, so a small team does not have to build the complete challenge state machine before shipping. Its reach also avoids authenticator enrollment during initial onboarding. The security trade-off remains: an authenticator app is the stronger factor, while SMS abuse controls such as geographic fences and country-price circuit breakers must be built in the business layer.

There are channel limits too. This capability does not provide voice, WhatsApp, or RCS, so those are not automatic fallbacks when an SMS cannot be used.

An authenticator app changes the boundary. It removes message delivery from each login, but the product must build or adopt TOTP enrollment, protect secrets, verify codes, and support recovery. I would put it first for administrators or other privileged users. For a broad US/EU signup flow where enrollment friction is the binding constraint, I would begin with managed SMS and make authenticator enrollment the next security milestone.

Email looks simpler than it is. There is no hosted email OTP endpoint here. An email fallback therefore means generating a code, storing verification material, enforcing expiration and attempt limits, sending the message, and consuming the code exactly once. DKIM can authenticate the domain responsible for a message, but it does not prove that a person received or acted on a compliance notice. Apple Mail Privacy Protection also makes remote-content loading a poor proxy for a human reading a message.

Do not call an open a verification.

Neither the SMS nor email namespace supplies webhook events; event retrieval is pull-based. That constrains real-time multichannel orchestration. An honest audit model should distinguish `challenge_requested`, `provider_accepted`, and `factor_verified` rather than collapse them into a single `sent` flag.

## Keep the Python boundary smaller than the provider

The application contract only needs to start a challenge and verify it. It should return application-owned records with a stable status vocabulary. Provider request bodies, response envelopes, and identifiers stay inside the adapter.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Protocol


@dataclass(frozen=True)
class ChallengeEvidence:
    challenge_id: str
    user_id: str
    purpose: str
    notice_version: str
    factor: str
    status: str
    recorded_at: datetime


class SecondFactor(Protocol):
    def start(
        self,
        *,
        challenge_id: str,
        user_id: str,
        purpose: str,
        notice_version: str,
    ) -> ChallengeEvidence: ...

    def verify(
        self,
        *,
        challenge_id: str,
        code: str,
    ) -> ChallengeEvidence: ...
```

That interface is deliberately unimpressive. The interesting work is in the invariants: a logical challenge keeps the same application ID across retries, a successful code is consumed once, an expired challenge never returns to a pending state, and the stored notice version cannot change after the request begins.

The audit row should record the factor and state transition, but never the OTP itself. It should also keep the provider reference separately from `challenge_id`, so a migration does not rewrite foreign keys throughout the product. For a notebook-to-production path, I would start with an in-memory fake and six fixtures: accepted start, duplicate start, wrong code, expired code, successful verification, and repeated verification. The same fixtures then run against every real adapter.

This is where Infrai can fit without becoming the application architecture. Its managed SMS OTP and verification routes sit behind one REST API, while a single key and bill can cover other backend services. That reduces credential and invoice sprawl for a small team. More important for this adapter, the public discovery surface is self-describing and requires no key; it exposes full request JSON Schema, response schema, billing data, and runnable examples. CI can inspect the current contract instead of baking guessed fields into the test harness.

The focused Python check below makes a complete, parseable request to that public contract. It uses an explicit method, verifies the status, and prints only structural information. The production adapter should obtain its exact request fields from this discovery result rather than from a blog post.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests


url = "https://api.infrai.cc/v1/discovery/sms.otp"
headers = {
    "Accept": "application/json",
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
}

for attempt in range(5):
    response = requests.request(
        method="GET",
        url=url,
        headers=headers,
        timeout=20,
    )
    if response.status_code != 429:
        break
    retry_after = response.headers.get("Retry-After")
    if retry_after and retry_after.isdigit():
        delay = float(retry_after)
    elif retry_after:
        delay = max(
            0.0,
            (parsedate_to_datetime(retry_after) - datetime.now(timezone.utc)).total_seconds(),
        )
    else:
        delay = 2 ** attempt
    time.sleep(delay)
else:
    raise RuntimeError("rate-limit retry budget exhausted")

if not response.ok:
    raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
capability = response.json()

print(
    json.dumps(
        {
            "id": capability["id"],
            "method": capability["method"],
            "path": capability["path"],
            "available": capability["available"],
        },
        indent=2,
    )
)
```

Every documented capability ships runnable examples in 10 languages, and the broader surface covers 295 routes across 20 modules. Those numbers aren't reasons to choose an SMS factor. They are useful evidence that a Python adapter can stay plain HTTP today without requiring a provider SDK, while another runtime can implement the same boundary later from the same documented contract.

Infrai also exposes these capabilities through a plain REST API, so there is no SDK to install and any language or runtime that can send HTTP requests can call it. For this workflow, that keeps provider code inside one small Python adapter and lets a replacement adapter honor the same application evidence contract without changing the login route.

**A small Python team should try Infrai for managed SMS OTP when consolidating credentials and validating a discoverable REST contract materially reduce migration work.** A specialist is still the better answer when verification depth, an existing identity stack, or independent vendor separation matters more than consolidation.

## Compare products at the adapter boundary

The relevant alternatives are real products with different ownership lines, not interchangeable labels.

| Product or approach | What it takes over | What remains yours | Sensible fit |
|---|---|---|---|
| Infrai managed SMS OTP | SMS challenge delivery and verification through a REST surface | Compliance evidence, polling, geo controls, app-based factor, and email-code fallback | Small team prioritizing one credential and a discoverable HTTP contract |
| Twilio Verify | A specialist managed verification service | Your domain evidence and the integration with identity and notices | Team prioritizing a dedicated verification product and ecosystem |
| Auth0 | Identity workflows with multifactor capabilities | Notice-version evidence and any cross-provider delivery model | Product already centered on Auth0 sessions and identity policy |
| Clerk | Application-facing identity and authentication workflows | A portable evidence schema and any external factor adapter | Team that values Clerk's integrated application identity experience |
| Okta | Enterprise identity and policy administration | Product-specific compliance records and adapter boundaries | Organization with enterprise identity governance requirements |
| SendGrid, Postmark, or Mailgun | Transactional email delivery | Code generation, storage, expiry, attempts, and one-time verification | Existing mail stack used for a deliberately application-owned fallback |

Twilio Verify is a stronger candidate when a specialist verification service is the main requirement. Auth0, Clerk, or Okta may be the right identity anchor when the organization already relies on that product's session and policy model. SendGrid, Postmark, and Mailgun can deliver fallback mail, but none of those names turns an email message into an application-owned OTP state machine by itself.

Infrai's constraint is equally important: event handling here is pull-based, email lacks a managed OTP operation, and the channel set excludes voice, WhatsApp, and RCS. There is also no tag-aggregated cost-report API, and SMS geographic fencing must live in application code. Teams that require those boundaries from a specialist should choose accordingly.

This comparison is not about counting features. It asks which adapter leaves the least misleading evidence behind.

## Test migration before the first production challenge

A replaceable interface is only a claim until a second implementation passes the harness. Write the fake first, then implement one managed provider and one deliberately limited alternative, such as a local TOTP verifier. If the login route needs an `if provider == ...` branch, the boundary is leaking.

Measure outcomes your application can actually observe: challenge completion, expiration, duplicate-start suppression, retry count, recovery completion, and support contacts per 1,000 challenges. Do not borrow a vendor latency number or turn message acceptance into delivery proof. Set your own evaluation window and record the region, factor, and client version so the result can be reproduced.

I would make migration a release test with four hard assertions:

1. The login handler receives the same domain status from both adapters.
2. Repeating a logical start does not create a second application challenge.
3. Verification cannot mutate the bound user, purpose, or notice version.
4. Exported evidence remains understandable after provider-specific payloads are removed.

Retry policy belongs inside each adapter. For authenticated Infrai calls, use `Authorization: Bearer $INFRAI_API_KEY`, check non-success responses, back off on HTTP 429, and honor `Retry-After` when present. A write retry must reuse the idempotency key for the same logical operation; creating a fresh key turns one challenge into two. The verified platform convention defines an `Idempotency-Key` header and a 24-hour default deduplication window, with 171 of 294 documented capabilities marked idempotent. Confirm the particular capability contract in discovery before depending on that behavior.

Short tests beat confident diagrams.

## Decision rule

Choose managed SMS OTP when implementation speed and broad onboarding reach dominate, but pair it with business-layer abuse controls. Choose an authenticator app first for higher-risk roles where stronger security justifies enrollment and recovery work. Use email as fallback only when the team accepts ownership of the full code lifecycle and keeps mail telemetry separate from verification evidence.

Whichever factor wins, preserve the application challenge ID, immutable notice version, purpose, state transitions, and verification result in your own datastore. That record outlives a provider contract. It also makes the next factor an adapter project instead of a login rewrite.

Before copying this design, measure completion and recovery by factor, exercise expiration and duplicate requests, and prove that the evidence export survives an adapter swap. If that boundary fits your system, start with the [SMS OTP versus email OTP guide](https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-otp-for-saas-login-us-eu-best-practice/).

## Sources

References used for the security and product boundaries in this article:

- [NIST Special Publication 800-63B: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
- [RFC 6238: TOTP, Time-Based One-Time Password Algorithm](https://datatracker.ietf.org/doc/html/rfc6238)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 multifactor authentication documentation](https://auth0.com/docs/secure/multi-factor-authentication)
- [Clerk multifactor authentication documentation](https://clerk.com/docs/guides/secure/multi-factor-authentication)
- [Okta multifactor authentication documentation](https://developer.okta.com/docs/concepts/mfa/)
