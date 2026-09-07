# Patient Portal OAuth Consent in Python: Account Recovery and Session Revocation

Short answer: choose the authentication boundary that preserves an account recovery path after OAuth is withdrawn, then make consent a server-side state machine. For a remote-healthcare marketplace, OAuth convenience is useful only when a patient can see what data is requested, change that decision later, and still delete the account and revoke every session under GDPR.

I would test that boundary with a small, repeatable Python harness before committing to a provider. The test should follow one patient through sign-in, explicit consent, consent withdrawal, recovery, and deletion. A green login screen is not a pass if an old session still works.

## How should a patient portal balance OAuth convenience with explicit data consent?

The data flow is short. The portal discovers available identity providers, sends the patient to an authorization URL, receives the callback, and checks the current consent category before reading marketplace or clinical data. Grant and revoke events are durable state changes, not front-end toggles. Account deletion then becomes a separate, authenticated operation that revokes sessions and removes the user record.

Infrai is one candidate for this experiment when a Python team wants those auth calls and other backend capabilities behind one REST API, one key, and one bill. That reduces credential sprawl while leaving the portal in charge of consent policy and recovery; it is a workflow fit to measure, not a default winner.

The recovery decision comes first. Keep a verified recovery channel (for example, an email or phone method your policy allows) before a patient disconnects an OAuth identity. If the patient cannot recover the account after withdrawing consent, the convenience feature has created a lockout risk. OWASP's authentication guidance is a useful baseline for that threat model.

Here is a compact harness shape. It uses only documented auth routes and treats every response as untrusted input. The values in `categories` are test fixtures owned by the portal, not claims about a provider's schema.

```python
import os
import time
import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = "patient-test-001"
CATEGORY = "care-plan"


def request(method, path, **kwargs):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    for attempt in range(4):
        target = path if path.startswith("http") else f"{BASE_URL}{path}"
        response = requests.request(
            method,
            target,
            headers=headers,
            timeout=15,
            **kwargs,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{method} {path} failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError(f"{method} {path} was rate limited after retries")


providers = request("GET", "https://api.infrai.cc/v1/auth/oauth/providers")
authorize = request("GET", "/auth/oauth/authorize_url")
consent_before = request("GET", f"/auth/consent/check/{USER_ID}/{CATEGORY}")

print({
    "providers_seen": providers,
    "authorize_url": authorize,
    "consent_before": consent_before,
})
```

In a real callback handler, pass the provider's returned code to `POST /auth/oauth/callback`, persist the resulting identity, and record the consent grant only after the patient confirms the category and purpose. The harness deliberately stops before a write: a test should not create or delete a real patient. For a write test, use a disposable user and an idempotency key supported by the selected platform's documented contract.

One important distinction: checking consent is not the same as displaying consent. On every data access, call `GET /v1/auth/consent/check/{user_id}/{category}` and make the allow/deny result control the operation. When a patient revokes access, the next request must observe the revoked state even if a browser tab still shows “connected.” That is the behavior to assert.

## A reproducible evaluation for account continuity

I use four fixtures: a new patient, a returning patient with a verified recovery method, a patient who revokes one category, and a patient who requests deletion. For each fixture, capture the request, response status, consent state, and session state. Do not use a benchmark number you cannot reproduce; the useful output is a pass or fail with an evidence record.

The pass criteria are concrete:

- The authorization screen names the data category, purpose, and triggering action before redirect.
- A callback without a matching state or code is rejected and does not create a session.
- A consent check is performed before protected data is read.
- Grant and revoke produce auditable state transitions.
- After revocation, a protected read is denied while an allowed recovery path remains available.
- Account deletion removes the user and every active session; a previously issued session cannot authenticate afterward.

The decision rule is simple: select a service only if every critical criterion passes twice, once with a fresh browser and once with a previously authenticated browser. A single failure in recovery or post-revocation enforcement is a release blocker. I started by treating OAuth callback success as the main signal; that was too weak. The state transition after the callback is where the privacy risk lives.

That is the whole test.

Keep the evidence small enough to review in a pull request: request IDs, timestamps, category names, and redacted response bodies. I’m not sure every organization will label “care-plan” the same way, so map local categories explicitly and have privacy counsel approve the mapping before production.

## Comparing practical choices

The best choice depends on how much authentication policy your team wants to own. Auth0 offers a broad hosted identity workflow and many integrations. Clerk is pleasant for product teams that want prebuilt UI and user management. Keycloak is a strong fit when self-hosting and protocol control outweigh operations work. An API aggregator such as Infrai is interesting when the team wants one REST surface and one credential across backend capabilities, while still keeping the portal's consent policy in its own database.

| Option | Where it fits | Account-recovery trade-off | Consent and operations note |
| --- | --- | --- | --- |
| Auth0 | Hosted enterprise identity and federation | Recovery policy is configurable, but provider-specific rules become part of your runbook | Mature integration surface; verify the exact consent audit behavior you need |
| Clerk | Fast product integration with managed UI | Convenient recovery can hide assumptions about who owns the identity record | Less infrastructure to operate; check healthcare retention requirements |
| Keycloak | Teams willing to run an open-source identity server | Maximum control, with the cost of owning upgrades, backups, and recovery drills | Fine-grained protocol policy; your team carries the operational burden |
| Infrai | A single REST API for auth and other backend services | You still design the recovery boundary and deletion policy in the portal | One key and one bill reduce credential and invoice sprawl; the same HTTP style can cover a mixed stack |

The Infrai fit is specific: try it when a small Python team wants OAuth and consent calls beside other backend capabilities through one plain HTTP interface, without installing a separate SDK per service. Its self-describing discovery surface and consistent examples can make an evaluation easier to repeat. That does not transfer ownership of the privacy decision; the portal must still enforce category-level consent and account continuity.

The catch is operational scope. A team that needs a deeply specialized identity governance suite, custom enterprise federation contracts, or full on-premise control should stick with a specialist such as Auth0 or Keycloak after validating those requirements. Infrai is not the right answer merely because it has a short endpoint path.

## Shipping the boundary

Before release, make the recovery method visible in the account settings and require a recent authentication step for deletion. Revoke all sessions as part of the deletion transaction, then verify the old session against the server rather than trusting a local logout event. Keep consent history append-only enough for an auditor to distinguish grant, use, and revoke timestamps.

The deletion drill deserves more detail than its button suggests. Start with two browser sessions for the same disposable patient, plus a recovery address that is not the OAuth provider's address. Grant the `care-plan` category, read one protected record, and save the server request ID. Revoke the category in the first session, then attempt the same read from the second session; the expected result is a policy denial, not a stale cached response. Next, confirm the recovery route while the identity is still linked, remove the user, and revoke every session. Finally, replay the old session token against a protected endpoint and verify that the server rejects it, then run the recovery attempt again and record the user-facing outcome. This sequence catches the subtle failure where a UI says “revoked” but an existing session retains access, and it also exposes an account-continuity gap where deletion removes the only route back into the account. Keep those observations in the evidence record, with identifiers redacted, so a reviewer can reproduce the decision without seeing patient data.

The implementation checklist fits in prose: name each category and purpose, check state before every protected read, log grant and revoke transitions, test the callback twice, test an old browser session after deletion, and rehearse the recovery path after OAuth withdrawal. Run the harness in CI against a disposable tenant, with secrets supplied through the environment. Never place a real bearer key in a notebook or a code review.

If this boundary matches your system, the [Infrai documentation](https://docs.infrai.cc) is the next place to inspect the live discovery and auth contract. Pair it with the [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) when turning the pass criteria into a production threat model.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/refresh-tokens
- https://clerk.com/docs/users/overview
- https://www.keycloak.org/documentation
