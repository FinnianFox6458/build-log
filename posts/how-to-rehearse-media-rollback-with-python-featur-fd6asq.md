# How to Rehearse Media Rollback with Python Feature Flag Kill Switches

A feature flag kill switch can stop a media application's next risky action while retaining evidence for a customer incident. Put the check immediately before the behavior you want to contain. Record its decision alongside the content revision. A switch cannot undo an article or notification that already left the system.

Short answer: use a dedicated flag for the AI publishing path, retain a decision record for each attempted action, and rehearse the disabled path before production. The evaluation constraint is rollback safety: disabling generation must preserve the last approved copy and explain which customer request used which decision. Incident response can be manual or driven by an application-owned workflow.

Infrai offers 295 routes across 20 modules under one key. One key, one bill: a single credential covers multiple backend services through a unified API. Its plain REST API works with any language or runtime; no SDK required. Adding a capability uses the same interface rather than another SDK integration. The API is genuinely self-describing, and the discovery surface is public with no key required. That helps a Python application move from notebook to production without guessing request fields.

## What can a feature flag kill switch roll back in production?

Start with the publication boundary. In the illustrative fixture below, a media service has an approved summary and a candidate AI summary. The kill switch controls which text the service selects for the next response. It does not erase a previous response, retract a notification, or reverse an external publication. Those operations need their own recovery procedures.

That distinction matters.

A tempting first design records only the flag's current value. During reconstruction, that leaves a specific ambiguity: the value you read after an incident may differ from the value used when a customer received a summary. Recording a decision at the action boundary closes that particular gap. Keep the flag key, its owner, the request identifier, the content revision, the observed decision, and the selected path together. An incident timeline can then connect a customer's request to an application action without treating today's configuration as historical evidence.

For a notebook-to-prod workflow, the useful artifact is a small executable rehearsal. It should establish what disabled means before any provider adapter enters the picture. The example uses synthetic media records, so it needs neither a customer transcript nor a live model call. The same decision boundary applies to a Node.js production service; this note uses Python for the rehearsal and API probe.

## Step 1 Define the evidence contract and run the rehearsal

Use a dedicated key such as `media.ai_summary.enabled`, with an owner who can explain its intended scope. Avoid letting the same key govern summary generation, newsletter delivery, and unrelated background work. Those boundaries have different recovery consequences, even if they share an incident commander.

The following example uses one local JSON file as a control-plane fixture and another as a decision journal. These files are teaching artifacts. They are not a production flag service or a tamper-resistant audit store. The record fields are application-defined; they make no assumptions about a vendor's response schema. The synthetic request IDs and revisions deliberately stay stable across each run.

Save this as `media_switch_rehearsal.py` and run it with Python 3. It uses only the standard library and writes inside a temporary directory. The assertions are the acceptance checks. A passing run prints the three evidence records for inspection; these are synthetic results, not a production benchmark.

```python
import json
import os
import tempfile
import time
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

FLAG_KEY = "media.ai_summary.enabled"
OWNER = "editorial-platform"


def inspect_remote_flag():
    origin = os.environ["INFRAI_API_ORIGIN"].rstrip("/")
    if not origin.startswith("https://") or origin.count("/") != 2:
        raise ValueError("Configure an HTTPS service origin without a path")
    request = Request(
        origin + "/v1/flags/is_enabled/{key}".format(key=FLAG_KEY),
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"Flag read failed with HTTP {error.code}: {reason}"
                ) from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after).timestamp()
                    delay = max(delay, deadline - time.time())
            time.sleep(delay)


def read_enabled(control):
    try:
        flags = json.loads(control.read_text(encoding="utf-8"))
        value = flags[FLAG_KEY]
        if type(value) is not bool:
            raise ValueError("Flag value must be boolean")
        return value, "observed"
    except (OSError, ValueError, KeyError, TypeError):
        return False, "unavailable_or_invalid"


def select_summary(control, journal, request_id, revision, approved, candidate):
    enabled, observation = read_enabled(control)
    evidence = {
        "request_id": request_id,
        "content_revision": revision,
        "flag_key": FLAG_KEY,
        "flag_owner": OWNER,
        "observed_enabled": enabled,
        "observation": observation,
        "selected_path": "candidate" if enabled else "approved",
    }
    with journal.open("a", encoding="utf-8") as stream:
        stream.write(json.dumps(evidence, sort_keys=True) + "\n")
        stream.flush()
    return candidate if enabled else approved


def rehearse():
    approved = "The newsroom has approved this summary."
    candidate = "A synthetic candidate summary for this rehearsal."
    with tempfile.TemporaryDirectory() as directory:
        root = Path(directory)
        control = root / "flags.json"
        journal = root / "decisions.jsonl"
        control.write_text(json.dumps({FLAG_KEY: True}), encoding="utf-8")
        first = select_summary(
            control, journal, "reader-001", "story-r17", approved, candidate
        )
        control.write_text(json.dumps({FLAG_KEY: False}), encoding="utf-8")
        second = select_summary(
            control, journal, "reader-002", "story-r17", approved, candidate
        )
        control.unlink()
        third = select_summary(
            control, journal, "reader-003", "story-r17", approved, candidate
        )
        records = [
            json.loads(line)
            for line in journal.read_text(encoding="utf-8").splitlines()
        ]
        assert first == candidate
        assert second == third == approved
        assert len(records) == 3
        assert [item["selected_path"] for item in records] == [
            "candidate", "approved", "approved"
        ]
        assert records[2]["observation"] == "unavailable_or_invalid"
        print(json.dumps(records, indent=2))


if __name__ == "__main__":
    if os.environ.get("INFRAI_API_ORIGIN"):
        print(json.dumps(inspect_remote_flag(), indent=2))
    rehearse()
```

Here the check happens before selection, and a journal write failure prevents the function from returning a selected summary. That is an explicit choice for this evidence-sensitive example. A production service must decide whether journal unavailability should block publication, buffer a durable record, or trigger another approved path. A casual background log write does not establish that guarantee. A successful file flush also does not make this local journal a crash-safe production ledger.

Keep that boundary visible.

The optional remote probe uses the verified flag-read route, with a timeout of 10 seconds per request and at most 4 attempts. To run it, configure `INFRAI_API_ORIGIN` with the service's HTTPS origin from your deployment configuration and `INFRAI_API_KEY` with your credential. Leave the origin unset for an entirely local rehearsal. The probe makes a read-only request, retries HTTP 429 up to 3 times with exponential backoff and Retry-After support, and prints the returned JSON without assuming an undocumented boolean field. It does not mutate a flag. Use the discovery response schema to implement a provider adapter before replacing the local fixture; this example keeps the network probe and the synthetic acceptance checks separate.

There is another limit to the rehearsal: the candidate text already exists. If the expensive or risky action is model generation, check the flag before invoking the model as well. Selecting approved text after generating a candidate can protect the displayed response while still consuming tokens. The eval-driven version tests both boundaries separately: no candidate publication when disabled, and no model invocation when generation is disabled. Count invocations rather than inferring them from which text appeared.

This is the prompt-cost aware part of the design. It doesn't require turning a rollback decision into a pricing argument.

## Step 2 Choose the control plane by the recovery obligation

A provider choice should follow the rehearsal's acceptance criteria. The comparison below distinguishes candidates to investigate from capabilities established by this local example. It is a selection guide, not a claim that these products passed a shared live test.

| Option | Useful role in this media workflow | Boundary to verify before choosing |
| --- | --- | --- |
| LaunchDarkly | A dedicated feature-management candidate when the decision is centered on flag operations | Verify the selected plan's history, approval controls, and Python evaluation behavior against the incident runbook |
| Unleash | Another dedicated feature-management candidate for a team comparing control-plane operating models | Verify deployment responsibilities, change evidence, and how the chosen client handles stale decisions |
| Statsig | A feature-management candidate when the team also wants to investigate experiment-related workflows | Verify that emergency containment has a clear owner and acceptance criteria independent of experiment analysis |
| Datadog | A logging and observability candidate for the evidence side of reconstruction | Log ingestion and indexing have a pricing model to review; collecting evidence does not itself specify the application's safe publication path |
| Sentry | A candidate to evaluate when reconstruction starts with an application error | Verify that the captured event can be joined to the publishing decision; error evidence alone cannot reverse a publication |
| Grafana | A candidate to evaluate when reconstruction starts with an operational timeline | Verify the underlying records and retention; a useful dashboard is not proof of which revision reached a reader |

Do not read the table as a ranking. The first three deserve a comparison when flag administration is the main requirement. Datadog belongs to the evidence discussion; it is not being presented here as an interchangeable flag evaluation API. Product documentation should settle specific plan and SDK questions before implementation.

The trade-off is explicit: the local file approach is a rehearsal, while the production choice needs an owned incident workflow, polling policy, and independent decision journal. Keep flag ownership and scope in application configuration. Preserve the observed value for each action, and test the time between an operator's change and the last worker observing it. Require an external evidence store with the retention, export, and deletion controls your incident process needs; those requirements deserve their own acceptance checks.

## Step 3 Measure containment before copying the choice

Start with the three synthetic decisions above. The expected selected paths are candidate, approved, approved. Then extend the same harness to count model calls, reject malformed control values, and verify that a missing journal prevents publication. The criterion is behavior under a disabled or unreadable flag, not whether the dashboard looks green.

One trap deserves a separate rehearsal: a worker may already have read an enabled value when an operator disables the flag. A check before generation and another before publication narrow different risks. Neither cancels an external action that has already happened. Record decision time and configuration observation time separately in a production design, then measure stale-decision age and in-flight publications during a controlled drill. No propagation time is established by this local example.

For customer reconstruction, join a request ID to the exact content revision and selected path. Keep the record free of API keys and unnecessary reader data; OWASP's logging guidance is a useful review checklist. If the journal is incomplete, report that gap rather than treating a current flag value as evidence of an earlier decision.

Rollback safety wins here. Choose the control plane only after the rehearsal proves the safe path, the incident workflow has an owner, and the evidence survives long enough to answer the customer's actual question.

## References

The sources below support the logging review and the product comparisons. Check current product documentation for plan-specific behavior; this note reports no shared vendor benchmark.

## Further reading

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- Datadog pricing and log ingestion/indexing model: https://www.datadoghq.com/pricing/
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Statsig documentation: https://docs.statsig.com/
- Sentry documentation: https://docs.sentry.io/
- Grafana documentation: https://grafana.com/docs/
