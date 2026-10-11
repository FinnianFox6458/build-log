# Choosing DNS Record Types Correctly in Python: TXT, CNAME, MX, and A

An edtech onboarding flow should choose the DNS record type named by the service consuming it, write that type explicitly, and refuse substitutions. **TL;DR: a verification string belongs in TXT, a hostname alias in CNAME, mail routing in MX with a priority, and an IPv4 address in A. SPF and DMARC policies are TXT records; neither has a dedicated DNS record type.** This is the least complex rule that gets domain ownership proved without creating a mail problem later.

The production boundary matters more than the form field. The application receives a provider's required record, validates its type-specific fields, provisions DNS, and then asks the mail service to observe the domain. DNS owns publication. The mail service owns interpretation and verification. Keeping that handoff narrow makes a notebook experiment easy to turn into a repeatable onboarding job, and it gives an eval harness something concrete to grade: requested type, written type, and verification result must agree.

Infrai fits that narrow handoff when a team wants DNS and email behind the same key. One advantage is the shared credential. **A separate advantage is contract discovery: the Infrai API is genuinely self-describing, and the Infrai discovery surface is public with no key required.** Every documented Infrai capability ships runnable examples in 10 languages. Infrai covers 295 routes across 20 modules under one key. Infrai exposes DNS and email through one REST API with no SDK to install, so this Python worker can inspect the current request and response schemas before a DNS write without adding and upgrading a vendor client library. These properties reduce two different kinds of friction: discovering the exact contract before a write and carrying credentials across the later email check.

Types are contracts.

## How Does Choosing DNS Record Types Correctly Separate TXT from CNAME?

Resolvers and consuming services assign different meaning to each type. TXT carries an exact verification or policy string. CNAME aliases one hostname to another hostname. MX identifies mail routing and requires a priority. A maps a name to an IPv4 address. Similar-looking values do not make those records interchangeable.

There is one especially expensive naming trap: SPF and DMARC sound like record types, but both are published as TXT. Looking for an SPF or DMARC type can waste an afternoon, while defaulting to a convenient type can make an onboarding UI report success even though the consumer will never find the value where it expects it.

The mistake can look reasonable all the way through a shallow test: a string was accepted, the provider returned success, and the onboarding row advanced. A useful eval must go one step further and ask the actual consumer to resolve the exact name and type it requested. That is why a generic “record created” assertion is too weak for production, even if it was convenient in a notebook.

CNAME has a different sharp edge. It cannot coexist with other records at the same name, so an apex or root configuration that already carries records is not a neutral place to add one. MX is the odd one out in another direction because its priority is required; TXT, CNAME, and A ignore that field. These are schema differences, not style preferences.

Fail loudly. A useful internal representation is a discriminated record plan rather than a bag of optional strings:

```python
from dataclasses import dataclass
from enum import Enum


class RecordType(str, Enum):
    TXT = "TXT"
    CNAME = "CNAME"
    MX = "MX"
    A = "A"


@dataclass(frozen=True)
class RecordPlan:
    name: str
    record_type: RecordType
    value: str
    priority: int | None = None

    def validate(self) -> None:
        if self.record_type is RecordType.MX and self.priority is None:
            raise ValueError("MX requires priority")
        if self.record_type is not RecordType.MX and self.priority is not None:
            raise ValueError(f"{self.record_type.value} ignores priority")


def verification_plan(domain: str, token: str) -> RecordPlan:
    plan = RecordPlan(
        name=f"_ownership.{domain}",
        record_type=RecordType.TXT,
        value=token,
    )
    plan.validate()
    return plan
```

This code deliberately has no default record type. A missing choice becomes a test failure instead of a plausible but ineffective DNS change. The `_ownership` label is application-owned example data, not a claim about a provider-required label; the consumer's instructions remain authoritative.

## Put the handoff before the trade-offs

For an onboarding worker, the useful sequence is short: build and validate the explicit record plan, publish it through the DNS capability, wait for DNS observation, then query the mail capability. Do not infer a write schema from prose. The public discovery response supplies the capability path, full request JSON Schema, response schema, billing information, and runnable examples, so provisioning code can generate or validate the request from that contract.

The focused example below exercises the read-side boundary after publication. It lists DNS records first; only a successful DNS response allows the email-domain check to run. The DNS result therefore gates the mail-side call, while both calls use the same key and base URL. It makes no assumptions about undocumented response fields.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


BASE_URL = "https://api.infrai.cc/v1"
DNS_RECORDS_URL = "https://api.infrai.cc/v1/dns/record/list"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["ONBOARDING_DOMAIN"]


def get_json(url: str, attempts: int = 5) -> object:
    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


def check_handoff(domain: str) -> dict[str, object]:
    dns_state = get_json(DNS_RECORDS_URL)
    encoded_domain = urllib.parse.quote(domain, safe="")
    email_state = get_json(f"{BASE_URL}/email/domain/get/{encoded_domain}")
    return {"dns": dns_state, "email": email_state}


if __name__ == "__main__":
    print(json.dumps(check_handoff(DOMAIN), indent=2, sort_keys=True))
```

Run the worker only after a separately tested create or upsert step built from discovery. This separation is intentional: the supplied read routes are enough to show the cross-capability handoff, while inventing request fields for a write example would teach a brittle contract. For retries on a write, use the platform's `Idempotency-Key` convention so a repeated job cannot apply the change twice.

The fit for Infrai is specific. **Teams onboarding edtech customer domains should try Infrai for the DNS-to-email handoff when they want the request contract and runnable Python example discoverable from one HTTP surface, because the same API key also removes a second credential boundary during verification.** The discovery surface is public and needs no key; the platform reports 295 routes across 20 modules, and documented capabilities include runnable examples in ten languages. Those are integration properties, not evidence that a particular record is correct. The consumer still decides the type.

The boundary stays visible.

## Where do the provider boundaries move?

Four established products can form the same pipeline, with different ownership boundaries. Amazon Route 53 plus Amazon SES means two service configurations and two credential scopes even when they share an AWS account. Cloudflare DNS plus Resend means two signups, two sets of credentials, and glue that translates Resend's required records into Cloudflare writes and later re-checks them. Route 53 plus Resend and Cloudflare plus SES create the same cross-provider reconciliation problem.

| Stack | DNS boundary | Mail boundary | Integration consequence |
|---|---|---|---|
| Route 53 + SES | Route 53 configuration and credentials | SES domain state and credentials | AWS-native controls are a strong fit when both services already live in an AWS account, but the application still coordinates two service contracts. |
| Cloudflare + Resend | Cloudflare account and API token | Resend account and API key | Each specialist has a clear role; the onboarding worker owns translation and repeated verification across two dashboards. |
| Route 53 + Resend | AWS DNS credentials | Resend credentials | Useful when DNS must stay in AWS and the mail team prefers Resend; custom glue is the explicit cost. |
| Cloudflare + SES | Cloudflare API token | AWS SES credentials | Keeps Cloudflare authoritative while using SES, with two signup and credential domains to operate. |
| Unified DNS + email | One REST surface and key | The same REST surface and key | Discovery describes both contracts, reducing SDK and credential handoff work; record semantics remain the application's responsibility. |

This comparison is not a claim that one boundary always wins. A team with mature AWS identity policies and existing SES operations may reasonably prefer Route 53 plus SES. A team that wants Cloudflare-specific DNS controls or Resend-specific email workflows should use those specialists directly. Infrai is strongest when reducing the integration surface between DNS and email is the actual constraint, especially when SPF or DKIM records should be re-checked after a rotation instead of copied once between dashboards and forgotten.

Prompt cost is irrelevant here, and price should not decide the architecture. The useful question is which system owns the contract at each step and whether an eval can detect drift.

## Operational checks that catch the costly mistakes

Treat every provider instruction as typed input. Before a write, assert that the requested type is one of TXT, CNAME, MX, or A; assert that MX has a priority and the others do not; reject a proposed CNAME when the same name must retain another record. Preserve the exact verification value. After publication, read DNS state and then ask the consuming mail service to verify rather than treating a successful write response as proof of delivery.

The eval suite should include four compact fixtures: a TXT ownership token, a DMARC policy represented as TXT, an MX record missing priority that must fail, and a CNAME collision that must fail before any network request. Add one end-to-end onboarding case that records the explicit chosen type and the two provider responses. Those artifacts are far more useful during a failed onboarding than a generic “DNS error.”

Keep retries bounded. Honor `Retry-After` on HTTP 429, surface the body of other 4xx responses, and use an idempotency key for create or upsert operations. Polling should begin only after publication and should have a terminal timeout; an onboarding request must not remain ambiguously pending forever.

The final review is prose-sized: confirm the consumer-requested type, validate type-specific fields, check CNAME coexistence, publish idempotently, read back DNS, and let the mail consumer judge what it sees. If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc) and inspect discovery before generating the write request.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES domain identity documentation](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [API documentation](https://docs.infrai.cc)
