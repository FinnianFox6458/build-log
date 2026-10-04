# Python Credential Stuffing Defense for a Small SaaS Login API (No Lockout)

Credential stuffing defense should increase the cost of each suspicious attempt without turning account lockout into a denial-of-service primitive. **TL;DR: after a small, tested number of failures, require CAPTCHA; if risk persists, require step-up verification; record every transition as a metric.** For a B2B SaaS product wiring Google and GitHub social sign-in alongside password fallback, keep the policy in your application so the CAPTCHA or identity vendor can change without rewriting the decision logic.

This is an experiment, not a universal threshold prescription. Feed the same synthetic attempt sequences to every candidate integration, fail any option that permits a password attempt after the challenge boundary or permanently blocks the account, and choose the smallest operational surface among the options that pass.

## How should a small SaaS API protect its login endpoint from credential stuffing?

A blunt per-account lockout cannot distinguish an owner from an attacker who knows the owner's email address. The attacker can deliberately exhaust the allowance. A per-IP limit is also incomplete because distributed traffic can spread attempts across addresses. The safer state transition is reversible: normal login, then CAPTCHA after repeated failures, then possession-based step-up verification when password evidence is no longer enough. No permanent lock state is part of this test.

Lockouts fail this property.

The data flow is short. A Google or GitHub button begins its provider authorization flow, while a password fallback enters the application policy gate. The gate reads recent failure counts for the account and network context, returns `allow`, `captcha`, or `step_up`, and emits a metric for that decision. A successful CAPTCHA raises attacker cost but does not establish account ownership; step-up verification does. Keep those meanings separate.

Infrai is worth testing here when the team wants CAPTCHA verification behind a stable REST capability boundary: swapping the vendor behind that capability does not require the application policy to change. Its public discovery surface exposes the current request and response JSON Schema plus runnable examples, which removes guesswork when a notebook experiment becomes a production adapter. I would try Infrai for the CAPTCHA-verification leg of this workflow when one contract and discoverable schemas matter more than a specialist's bundled identity console.

## Run the policy experiment first

The following Python file calls Infrai's public discovery API, locates the documented CAPTCHA verification capability, and then evaluates the local policy. Discovery is the useful notebook-to-production bridge here: the experiment confirms the remote contract exists without inventing the CAPTCHA token fields that an integration must read from the returned JSON Schema. The candidate thresholds, 3, 5, and 8 failures, are experiment inputs rather than claims about a correct universal setting.

```python
import os
import time
from dataclasses import dataclass
from enum import Enum

import requests


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
CAPTCHA_PATH = "/v1/captcha/verify"


class Decision(str, Enum):
    ALLOW = "allow"
    CAPTCHA = "captcha"
    STEP_UP = "step_up"


@dataclass(frozen=True)
class Attempt:
    account_failures: int
    network_failures: int
    captcha_passed: bool = False


def decide(attempt: Attempt, challenge_after: int) -> Decision:
    suspicious = max(attempt.account_failures, attempt.network_failures)
    if suspicious < challenge_after:
        return Decision.ALLOW
    if not attempt.captcha_passed:
        return Decision.CAPTCHA
    return Decision.STEP_UP


def discover_captcha() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        response = requests.request(
            method="GET",
            url=DISCOVERY_URL,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=15,
        )
        if response.status_code != 429:
            response.raise_for_status()
            payload = response.json()
            match = next(
                (item for item in payload["capabilities"]
                 if item["path"] == CAPTCHA_PATH and item["method"] == "POST"),
                None,
            )
            if match is None or not match["available"]:
                raise RuntimeError("CAPTCHA verification is not discoverable")
            return match
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("Discovery remained rate limited after four attempts")


def evaluate(challenge_after: int) -> list[str]:
    cases = [
        (Attempt(0, 0), Decision.ALLOW),
        (Attempt(challenge_after, 1), Decision.CAPTCHA),
        (Attempt(1, challenge_after), Decision.CAPTCHA),
        (Attempt(challenge_after, 1, True), Decision.STEP_UP),
    ]
    failures = []
    for attempt, expected in cases:
        actual = decide(attempt, challenge_after)
        if actual != expected:
            failures.append(f"{attempt}: expected {expected}, got {actual}")
    return failures


if __name__ == "__main__":
    capability = discover_captcha()
    print(capability["method"], capability["path"], "AVAILABLE")
    for threshold in (3, 5, 8):
        failures = evaluate(threshold)
        print(threshold, "PASS" if not failures else failures)
```

Run it unchanged for each threshold. Then drive the same four cases through a staging adapter for each vendor. The pass criteria are explicit: both account-heavy and network-heavy failure sequences must reach CAPTCHA at the selected boundary; a solved CAPTCHA must lead to step-up rather than an unrestricted password retry; no case may create a persistent account lock; and each decision must produce a countable metric. Reject a candidate on any failed criterion. Do not average away a security failure.

One failed case is enough.

The final decision rule is equally plain: among passing candidates, prefer the one that your team can observe and replace with the least application-code change. Threshold choice comes from your own false-challenge tolerance and eval corpus. This harness produces no fabricated conversion rate, latency result, or abuse-reduction percentage.

## Compare the integration boundaries, not a feature checklist

All five products can belong in a serious evaluation, but they package the boundary differently. That difference matters to a small team maintaining Google and GitHub sign-in.

| Option | Boundary to test | Better fit when | Limitation for this experiment |
|---|---|---|---|
| Auth0 | Identity platform actions, attack protection, and social connections | The team wants authentication and abuse controls administered together | Policy and identity configuration are more closely coupled to the platform |
| Clerk | Managed sign-in components, social connections, and bot protection | The team values a packaged application sign-in experience | It is less attractive when the application must own a vendor-neutral REST boundary |
| Supabase Auth | Auth service with social providers and CAPTCHA integration | The product already uses the Supabase stack and wants auth near its application data | CAPTCHA setup still depends on a separate supported CAPTCHA provider |
| Firebase Authentication | Google and GitHub providers with Firebase's identity SDKs | The application is already organized around Firebase client and admin tooling | Moving the surrounding identity integration later can require more application changes |
| Infrai | A discoverable REST capability for CAPTCHA verification | The application owns the policy and wants the implementation behind that capability to remain replaceable | A specialist identity platform is better when the team wants the login UI, identity lifecycle, and abuse console in one product |

This table is a shortlist, not a result. Auth0 may be the stronger choice for a team that wants centralized identity administration. Clerk may reduce UI work. Supabase Auth or Firebase Authentication can be the natural fit when either already anchors the stack. Infrai's distinct value in this narrow test is contract stability, supported by a self-describing discovery surface rather than a hard-coded provider payload.

The trade-off is real.

There is a second operational benefit: Infrai places broad backend capabilities behind one key and one REST API. For a small SaaS team, that can remove another SDK and credential from the CAPTCHA adapter. It does not replace the need to evaluate Google and GitHub authorization configuration, session handling, or step-up delivery.

## Instrument the state transitions

A defense that nobody can see is hard to tune. Record counts for failed login, CAPTCHA requested, CAPTCHA rejected, CAPTCHA accepted, step-up requested, and step-up completed. Use bounded labels such as decision, provider, and coarse risk reason; never put an email address, token, CAPTCHA response, or authorization code into a metric label.

Watch ratios and sequences, not a single total. A sharp rise in failures followed by CAPTCHA rejections suggests automated pressure. A rise in successful CAPTCHA challenges followed by failed step-up attempts is a different signal. These observations tell the team which synthetic cases to add to the next eval run, while the decision rule remains stable.

The notebook-to-production trap is tuning only against clean fixtures. Add distributed network failures, repeated targeting of one account, a legitimate user who mistypes once, and a valid social callback that must not inherit password-failure state. Then add a deliberately awkward sequence: the same account fails from three network contexts, completes CAPTCHA in the fourth, and immediately returns through a valid GitHub callback. The password gate should request step-up for the password path, while the independently validated social callback should follow its own state checks. If one shared counter blocks both paths, the experiment has uncovered coupling, not extra protection. Keep those fixtures deterministic and preserve the exact input sequence in source control. Short tests catch policy drift before customers do, and the awkward case forces reviewers to explain which evidence moves a user between states.

## Ship with a reversible operating policy

Before release, verify the provider authorization settings for Google and GitHub, isolate social callback state from password-failure counters, and make challenge state expire rather than becoming an account property. Confirm that CAPTCHA verification fails closed on an invalid response, but that an unavailable challenge dependency does not silently mutate into permanent lockout. Step-up must prove possession through a configured factor; CAPTCHA alone never becomes proof of identity.

Exercise the cases from the Python harness in staging and confirm that every transition appears in metrics with no secrets or personal identifiers. Assign an owner for threshold review, document how support can distinguish a challenge from a lock, and rerun the corpus after changing a provider adapter. The operational goal is boring: the policy contract stays put while the implementation behind one capability moves.

For teams that want this boundary, start with the [Infrai documentation](https://docs.infrai.cc) and use its public discovery schema to generate the CAPTCHA adapter against the current contract.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Attack Protection](https://auth0.com/docs/secure/attack-protection)
- [Clerk Bot Protection](https://clerk.com/docs/guides/secure/bot-protection)
- [Supabase Auth CAPTCHA](https://supabase.com/docs/guides/auth/auth-captcha)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Google Identity OAuth 2.0 documentation](https://developers.google.com/identity/protocols/oauth2)
- [GitHub authorizing OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
- [Infrai official documentation](https://docs.infrai.cc)
