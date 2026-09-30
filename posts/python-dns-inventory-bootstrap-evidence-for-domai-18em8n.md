# Python DNS Inventory Bootstrap: Evidence for Domains Predating Logistics Automation

Treat the first scan as evidence, not desired state. For a logistics domain that existed before the automation system, onboarding should proceed only after the team has captured repeatable DNS answers, preserved raw observations, and separated missing data from failed collection. **The decision rule is simple: no convergence until the evidence set is reviewable.**

TL;DR: bootstrap the known domain portfolio from business records, observe a narrow set of DNS names without changing them, record answers and collection errors with timestamps, and require a human to resolve ownership and mail-authentication gaps. Then import the approved snapshot and automate only the reviewed state. This makes deliverability evidence part of onboarding rather than an assumption hidden inside deployment.

## How should you bootstrap a DNS inventory for domains that predate automation?

A zone inventory is not the same thing as a zone transfer or a dump of every possible name. Here, proof means a bounded observation another engineer can inspect: which fully qualified name was queried, which record type was requested, when the query ran, what answer data came back, and whether collection failed. The boundary matters because DNS has no general operation for enumerating every record in an arbitrary public zone.

Start the domain list outside DNS. In a logistics company, inputs can include the legal domain register, tracking and booking hostnames, email-sending domains, carrier-integration configuration, certificate inventories, and domains declared by teams during onboarding. Reconcile those inputs into a candidate set before querying anything. DNS can confirm observations about a candidate; it cannot reveal an unknown domain.

Discovery comes first.

For deliverability, collect apex MX and TXT answers, the DMARC TXT answer at `_dmarc.<domain>`, and any DKIM selector names supplied by the mail owner. Do not guess selectors and interpret an empty guess as proof that DKIM is absent. DMARC policy discovery uses DNS TXT records and has explicit handling rules in RFC 7489, so both the queried name and returned policy need to remain visible.

Keep infrastructure context too: apex A and AAAA answers, NS, SOA, CAA, and explicitly declared service hostnames. That is seven apex record types in the example below, chosen to expose delegation, address, mail, authority, certificate-policy, and text evidence without pretending to enumerate the zone. This does not prove that a shipment-notification message reached an inbox. It gives reviewers evidence for distinguishing mail configuration from a broader delegation or resolution problem. Consider a domain used by both a booking portal and shipment notifications: an intact A answer says nothing about its MX path, while a DMARC TXT answer says nothing about whether the booking hostname still belongs to the same team. One domain can carry several operational responsibilities, and the bootstrap review must preserve those distinctions instead of assigning a single vague status such as “active.”

## Capture a reviewable snapshot with Python

The data flow is deliberately small. A versioned input file supplies domains and known DKIM selectors; a read-only collector asks a configured recursive resolver for selected record types; newline-delimited JSON preserves one observation per query; and a later review turns accepted observations into managed state. Collection and mutation are different jobs.

This focused example uses `dnspython`. It records positive answers, DNS-level negative answers, and operational errors separately. One row is emitted for every attempted query, preventing silence from being mistaken for success.

```python
from __future__ import annotations

import argparse
import json
from dataclasses import asdict, dataclass
from datetime import datetime, timezone
from pathlib import Path

import dns.exception
import dns.resolver


APEX_TYPES = ("A", "AAAA", "MX", "NS", "SOA", "CAA", "TXT")


@dataclass(frozen=True)
class Observation:
    domain: str
    name: str
    record_type: str
    observed_at: str
    status: str
    answers: list[str]
    error: str | None


def query(resolver, domain: str, name: str, record_type: str) -> Observation:
    observed_at = datetime.now(timezone.utc).isoformat()
    try:
        answer = resolver.resolve(name, record_type, search=False)
        values, status, error = sorted(item.to_text() for item in answer), "answer", None
    except dns.resolver.NXDOMAIN:
        values, status, error = [], "nxdomain", None
    except dns.resolver.NoAnswer:
        values, status, error = [], "no_answer", None
    except (dns.resolver.NoNameservers, dns.exception.Timeout) as exc:
        values, status, error = [], "collection_error", type(exc).__name__
    return Observation(domain, name, record_type, observed_at, status, values, error)


def names_to_query(domain: str, selectors: list[str]) -> list[tuple[str, str]]:
    names = [(domain, record_type) for record_type in APEX_TYPES]
    names.append((f"_dmarc.{domain}", "TXT"))
    names.extend((f"{selector}._domainkey.{domain}", "TXT") for selector in selectors)
    return names


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("input", type=Path)
    parser.add_argument("output", type=Path)
    args = parser.parse_args()
    inventory = json.loads(args.input.read_text(encoding="utf-8"))
    resolver = dns.resolver.Resolver(configure=True)
    resolver.lifetime = 5.0

    with args.output.open("w", encoding="utf-8") as stream:
        for item in inventory:
            domain = item["domain"].rstrip(".").lower()
            for name, record_type in names_to_query(domain, item.get("dkim_selectors", [])):
                row = query(resolver, domain, name, record_type)
                stream.write(json.dumps(asdict(row), sort_keys=True) + "\n")


if __name__ == "__main__":
    main()
```

A minimal input keeps discovery claims explicit rather than burying them in code:

```json
[
  {"domain": "parcel-status.example", "dkim_selectors": ["dispatch", "returns"]},
  {"domain": "freight-booking.example", "dkim_selectors": []}
]
```

Run the collector from a controlled environment and retain its dependency lock file beside the output. The five-second lifetime is a collection bound, not evidence that an unanswered query represents a missing record. Short. Clear. A timeout remains `collection_error` and must be retried or investigated.

Silence is not absence.

## Separate absence, failure, and disagreement

The most dangerous normalization step is collapsing every non-answer into `null`. `NXDOMAIN` says the queried name does not exist according to the DNS response. `NoAnswer` means the response did not contain the requested record type. A timeout or unavailable nameserver says the collector obtained no usable evidence. Those states lead to different review actions.

Resolver viewpoint creates a concrete trade-off. One recursive resolver produces a consistent run, but may hold cached data and represents only one observation path. Multiple viewpoints improve disagreement detection while adding queries, storage, and review noise. For a high-risk mail cutover, repeat collection after the relevant TTL interval and from the resolver viewpoints your organization has chosen. Do not invent a universal waiting period; retain answer TTLs if the evidence policy needs exact scheduling. The example omits TTL serialization to stay focused, so add it before using TTL-driven comparisons.

Record ordering also needs care. DNS answer order is not a dependable change signal. The collector sorts textual answers so a reorder does not create noise, while leaving content intact. For MX records, the preference remains part of the text. For TXT, preserve the resolver library's representation and parse mail policies in a separate validation stage rather than rewriting evidence during capture.

There is a tempting shortcut: generate desired records from an onboarding template and compare them with one live query. I would reject it because the template becomes the source of truth before ownership is established. The initial snapshot should be descriptive. Policy comes next.

Do not automate ambiguity.

## Turn observations into a controlled baseline

A useful review joins every candidate domain to an accountable owner, business purpose, delegation contact, mail owner, and evidence status. Ownership is organizational data; DNS cannot supply it. Distinguish customer tracking, shipment notifications, booking portals, carrier callbacks, redirects, defensive registrations, and domains believed retired. That classification determines the cost of a mistaken change.

The approval gate should compare two snapshots and highlight content changes, new collection errors, and unresolved ownership. A positive DMARC observation is evidence of a published record, not proof of end-to-end deliverability. DMARC evaluates authentication and alignment as defined by the standard; inbox placement depends on evidence outside this inventory. Keep mail-send tests linked to the review without merging their results into DNS facts.

This is where an eval-driven habit pays off. Build fixtures for `answer`, `nxdomain`, `no_answer`, timeout, multiple MX values, and split TXT strings; run them whenever the collector or parser changes. The fixture set is small and deterministic. Prompt or model calls add no value to collection, so they do not belong in this path. If a model later summarizes reviewer notes, keep its output outside the authoritative snapshot and evaluate that workflow separately.

After review, translate only accepted records into the automation schema. Preserve the immutable capture as provenance, record the reviewer and approval time, and expose the first planned mutation as a diff. **Automation begins after the baseline is agreed, not when the first query succeeds.**

## Operate the handoff without losing evidence

Before enabling changes, rerun collection for the approved set and stop if evidence has materially changed. Confirm every production domain has an owner, every collection error is resolved, mail owners supplied rather than guessed DKIM selectors, and planned state was reviewed against DNS observations and separate delivery tests. Store the raw snapshot, normalized comparison, approval record, and deployment diff under one change identifier.

During rollout, use the smallest domain cohort that matches operational risk. Observe DNS answers after changes from the chosen resolver viewpoints, then execute the existing mail-delivery checks for notification traffic. A syntactically present TXT record is not a successful shipment email. Rollback criteria should name both DNS divergence and failed delivery evidence, with an accountable person able to halt the cohort.

Once the first cohort is stable, expand deliberately. Schedule recurring observation to detect drift, but keep detection read-only: an unexplained difference should open review rather than overwrite live DNS or the approved baseline.

The finished inventory is modest: candidate provenance, observed DNS data, explicit errors, ownership, approval, and linked deliverability evidence. It is enough to move an old domain into automation without pretending a fresh scan can reconstruct its history.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc1035
- https://dnspython.readthedocs.io/en/stable/resolver-class.html
