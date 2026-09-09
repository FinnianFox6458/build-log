# Loyalty Account Deduplication: Python Identity Resolution in 3 Decisions Explained

Short answer: resolve identity before creating a loyalty user, and treat a device fingerprint as risk evidence rather than a primary key. A small Python gate that returns `new`, `review`, or `merge` is easier to evaluate and safer against bots than letting every signup become an account.

## Why duplicate loyalty accounts are an abuse problem

In a B2B SaaS loyalty system, duplicate accounts are not a cosmetic data-cleanup task. They can split a member's points, let an attacker farm welcome rewards, or make a legitimate customer look like a bot. The difficult part is that the same person can arrive with a new browser, a shared corporate network, or a privacy-preserving device setting.

Device fingerprints help, but they are probabilistic. A fingerprint can be shared by a whole office, reset by a browser update, or intentionally changed by an abuse script. Email, verified phone, organization membership, and prior login history each carry a different kind of evidence. The gate should combine those signals and preserve why it reached a decision.

I first thought a unique email constraint solved deduplication. It only solved one database collision. The real decision happens before the insert, while the request still has enough context to challenge a suspicious enrollment. That's the boundary I keep in a notebook prototype before moving it into a service.

Stop there.

## How should Python resolve identity before user creation?

The flow is deliberately boring: normalize the submitted claims, look up candidates, score independent signals, and only then provision a user. Keep raw evidence separate from the final score so an evaluator can replay the decision without calling a live service.

```python
from dataclasses import dataclass
from enum import Enum
import re


class Decision(str, Enum):
    NEW = "new"
    REVIEW = "review"
    MERGE = "merge"


@dataclass(frozen=True)
class Signup:
    email: str
    phone: str | None
    org_id: str | None
    device_id: str | None
    ip_prefix: str | None


def normalize_email(value: str) -> str:
    local, domain = value.strip().lower().split("@", 1)
    local = re.sub(r"\+[^@]+$", "", local)
    return f"{local}@{domain}"


def resolve_identity(signup: Signup, candidate: dict | None) -> tuple[Decision, dict]:
    """Return an auditable decision before any user row is created."""
    if candidate is None:
        return Decision.NEW, {"reasons": ["no_candidate"]}

    reasons: list[str] = []
    score = 0
    if normalize_email(signup.email) == candidate["email"]:
        score += 60
        reasons.append("verified_email")
    if signup.phone and signup.phone == candidate.get("phone"):
        score += 25
        reasons.append("verified_phone")
    if signup.org_id and signup.org_id == candidate.get("org_id"):
        score += 10
        reasons.append("same_org")
    if signup.device_id and signup.device_id == candidate.get("device_id"):
        score += 5
        reasons.append("same_device")

    if score >= 80:
        decision = Decision.MERGE
    elif score >= 50:
        decision = Decision.REVIEW
    else:
        decision = Decision.NEW
    return decision, {"score": score, "reasons": reasons}
```

The thresholds are policy, not truth. In an eval harness, I would label a few hundred synthetic cases: returning-member changes email, two employees share a laptop, and a bot rotates fingerprints while reusing a phone. Then I would measure false merges separately from missed duplicates. A single accuracy number hides the harm of merging two real people.

First, decide which claims are strong enough to merge automatically. A verified email plus a verified phone may justify merging points, while a device match alone should never do that. Second, decide what happens when signals disagree: hold the enrollment for review, ask for step-up authentication, or create a quarantined record with no reward balance.

Third, decide how identity changes over time. Store a versioned evidence record, not just `matched_user_id`. That makes a later policy change explainable and lets support undo an incorrect merge without deleting an audit trail.

Here is the trade-off I use when choosing the enforcement point:

| Approach | Integration shape | Good fit | Main limitation |
| --- | --- | --- | --- |
| Database uniqueness | SQL constraint | Exact claims such as verified email | Too late for risk signals; cannot explain a review |
| Synchronous resolver | Python service call | Reward-bearing signup with step-up checks | Adds latency and an operational dependency |
| Asynchronous review queue | Event plus worker | High-value accounts and ambiguous matches | A member waits; queue ownership is real work |

Your mileage may vary. The right choice depends on how costly a false merge is compared with a delayed enrollment.

The catch is latency. A synchronous lookup across several identity stores can turn signup into a slow path, and a strict gate can reject a real customer behind a company NAT. If review staffing is small, choose conservative auto-merge rules and accept more manual work. If the business cannot tolerate a queue, use a lower-risk welcome reward that vests after a successful login.

## Where common identity services fit

Managed identity products can provide authentication, but they do not remove the domain decision about loyalty ownership. Auth0, Okta, and Amazon Cognito each expose different hooks, profile models, and token claims; your matching policy still needs a stable internal evidence schema. A self-hosted OIDC provider gives more control, while increasing patching and incident responsibility. I would not pretend the migration cost is identical across them: event names, profile mutability, and tenant boundaries vary, so write an adapter and test it against recorded claims.

Do not select a service because it has the longest feature list. Check whether it can export the claims you need, enforce phishing-resistant step-up methods, and emit events quickly enough for your enrollment gate. The right boundary is the one your eval cases can exercise in CI.

## An operational checklist in prose

Before launch, replay normalized fixtures through the resolver and pin the decision plus reasons in an append-only log. Include cases such as a shared tablet used by 12 warehouse staff, an address with a plus tag, and a device identifier that changes after an app reinstall; these are test fixtures, not assumptions about a real customer. Add counters for new, review, merge, and reversal outcomes; alert on a sudden rise in merges from one device prefix. Rate-limit enrollment, require reauthentication for a high-value merge, and make support reversals idempotent. OWASP's Authentication Cheat Sheet is a useful baseline for authentication controls, but your abuse policy remains application-specific.

The resolver should be boring to operate. Every outcome needs a reason code, a trace identifier, and a retention rule for the underlying claims. If legal or privacy requirements prohibit storing a raw fingerprint, retain a keyed digest and its expiry instead. I’m not sure any universal retention window exists; your data-protection counsel and abuse-loss model should settle that number.

This approach is not suitable when you need anonymous, instant checkout with no durable member identity. In that case, stick with a temporary guest ledger and reconcile only after a verified claim. It is also a poor fit for teams that cannot operate an audit queue; a simpler email-only rule may be safer than an elaborate model nobody monitors.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://openid.net/specs/openid-connect-core-1_0.html
- https://www.rfc-editor.org/rfc/rfc9457

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://openid.net/specs/openid-connect-core-1_0.html
