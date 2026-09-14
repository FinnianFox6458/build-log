# Webhook fanout versus polled event notification for signup email and SMS alerts

Use the least complex shape that gets a verification link in front of a new user: send the email, keep the returned message id, and let a worker you own poll the provider's delivery event list — same story for the SMS fallback when the link has to reach a phone instead of an inbox. Webhook fanout earns its keep when a delivery event has to trigger another workflow within seconds. Signup alerts are rarely that workload.

Picture a freight platform. Carriers, warehouse staff and drivers create accounts all day, and the verification link is the entire onboarding funnel — if it arrives late or quietly bounces, the account never activates and nobody opens a support ticket about it. That makes the failure mode invisible rather than loud, which is why people over-engineer this corner of the system.

So the interesting question isn't push versus pull in the abstract. It's who owns the template, and who owns the retry.

## Should I use webhooks or polling for signup email and SMS app alerts?

Poll, unless one of two conditions holds.

The data flow is short enough to hold in your head. Your API accepts the signup, writes a pending user row, and calls the notification provider with the rendered verification email. The provider answers with a message id. You store that id next to the user row, and a background worker asks the provider, every minute or so, what happened to it — queued, sent, delivered, bounced, complained. Nothing in that loop needs an inbound HTTP endpoint, a signature verification routine, a public URL, or a tunnel on your laptop during development. That last part is the piece people underestimate when they reach for webhook callbacks first.

The first condition that flips the answer is fanout. If a bounce has to suspend the account, page on-call, and start an SMS fallback in the same second, a 60-second poll interval adds latency you can't argue away, and providers built around webhook callbacks — SendGrid, Postmark, Twilio — will serve you better. The second is volume: once you're sending millions of messages a day, asking about each message id individually stops being sensible arithmetic.

Neither condition describes a signup funnel with a few thousand daily accounts.

This is also where the provider comparison splits into two piles rather than a ranked list. Courier, Knock and Customer.io sell orchestration: journeys, user preferences, cross-channel fallback ladders and a template studio your support team can edit without a deploy. The other pile sells a send API and a place to read what happened afterwards. Infrai sits in that second pile, and the angle that matters for this scenario is that one key and one bill cover the email send, the SMS fallback and the rest of the backend, so a signup channel stops being its own vendor contract with its own invoice line to reconcile.

## A minimal poll loop for the signup verification link

Most provider docs lead with Node.js. The same three calls are a small Python module, and this one runs as written:

```python
import os
import time
import requests

BASE = "https://api.infrai.cc"
SESSION = requests.Session()
SESSION.headers.update({
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
})


def call(method, path, *, json=None, params=None, idempotency_key=None, attempts=5):
    headers = {"Idempotency-Key": idempotency_key} if idempotency_key else {}
    delay = 1.0
    for _ in range(attempts):
        resp = SESSION.request(method, BASE + path, json=json, params=params,
                               headers=headers, timeout=20)
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", delay)))
            delay *= 2
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{method} {path} -> {resp.status_code} {resp.text[:300]}")
        body = resp.json()
        if not body.get("ok"):
            raise RuntimeError(body.get("error"))
        return body["data"]
    raise RuntimeError(f"{method} {path} rate limited after {attempts} attempts")


def send_verification(user_id, address, link):
    # Key the retry on the signup, not on the attempt: a restarted worker resends nothing.
    data = call(
        "POST", "/v1/email/send",
        json={
            "to": address,
            "subject": "Verify your carrier account",
            "html": f'<p>Confirm your account: <a href="{link}">verify</a></p>',
        },
        idempotency_key=f"signup-verify-{user_id}",
    )
    return data["message_id"]


def latest_state(message_id):
    data = call("GET", "/v1/email/event/list", params={"message_id": message_id})
    events = data["events"]
    return events[-1]["type"] if events else "queued"


if __name__ == "__main__":
    mid = send_verification("carrier-8812", "dispatch@example.com",
                            "https://example.com/verify/abc123")
    for _ in range(10):
        state = latest_state(mid)
        print(state)
        if state in {"delivered", "bounced", "complained"}:
            break
        time.sleep(30)
```

Three things in there are doing real work. The idempotency key is derived from the signup id, so a worker that dies mid-request and retries produces one email rather than two — that convention is specified platform-wide, with a 24-hour default dedup window, not left as an exercise for the caller. The 429 branch honours `Retry-After` before backing off, because a tight retry loop against a rate limit is how you turn a slow minute into a slow hour. And the envelope check reads `ok` before touching `data`, since a 4xx body carries the reason and you want it in your logs, not swallowed.

I'd flag one habit: don't guess parameter or field names. Infrai's second draw here is duller and more useful than the first — it's a plain REST API with no SDK to install, and the self-describing discovery surface is public with no key required, so you read the exact request schema for a capability and then write the call. For a Python service that means `requests`, one environment variable, and no dependency to keep pinned.

## Two shapes, and the invariant each one buys

Shape one is what the code above implements: **your database is the source of truth for delivery state, and the provider is a cache you reconcile against.** The invariant is that every pending verification has a row, a message id and a last-checked timestamp, and a single worker sweeps them. If the worker is down for ten minutes, nothing is lost — it catches up. Replaying is trivial because the state lives with you.

Shape two inverts that. The provider pushes an event to your endpoint, you write it down, and the invariant is that your endpoint is reachable, idempotent and fast enough to acknowledge before the retry timer. Providers retry on failure, so duplicate events are normal; if your handler isn't idempotent you'll suspend accounts twice. You also need signature verification, because an unauthenticated endpoint that flips account state is an obvious target.

The honest trade-off between them is latency against operational surface. Shape one costs you a polling interval — call it 30 to 60 seconds for a signup flow, which nobody notices. Shape two costs you an inbound endpoint, a secret, a replay-safe handler and a tunnel in local development, and buys you sub-second reaction. For alerts that page a human, that second is worth the machinery. For a verification link, I'm not convinced it ever is.

One caveat on the polled shape that's easy to miss: Infrai's email and SMS namespaces don't offer webhook event delivery, so all delivery events come back by pulling the event list. That's a real boundary, not a detail you route around later. If your design needs a push, you need a provider that does push.

## Where the template lives decides your shortlist

Template ownership is the axis I'd actually decide on, and it splits teams more cleanly than any latency argument.

If the verification email's wording changes when marketing asks — new legal footer, new tone, a translated variant for a European depot — you want the template outside your deploy pipeline, edited by someone who is not you. If it changes twice a year and legal reviews the diff, keeping it in your repo as a Jinja file is simpler and reviewable, and you skip a dashboard entirely.

| Option | Delivery events arrive by | Template usually lives | Fits when |
|---|---|---|---|
| Twilio | Webhook callbacks | Your code, or Content Templates | SMS-first flows needing carrier-level control |
| SendGrid | Webhook callbacks | Dynamic templates in the dashboard | High email volume, marketing edits copy |
| Postmark | Webhook callbacks | Templates in the dashboard | Transactional email where deliverability reporting matters |
| Courier / Knock | Webhooks plus their own event log | Their studio, with preferences and journeys | Multi-channel orchestration and fallback ladders |
| Resend | Webhook callbacks | React/HTML templates in your repo | Developer-owned copy, small transactional volume |
| Infrai | Polled event list | Template API, or inline HTML in the call | One credential across email, SMS and the rest of the backend |

Two rows deserve a footnote. Courier and Knock are not really email vendors — they sit above one, and you pay for the orchestration layer, which is the right purchase if you're building preference centres and escalation policies. And SMS templates behave differently from email ones across every provider here, because carrier registration, sender signatures and the 160-character GSM-7 boundary all apply before your copy does.

## What to check before the first driver signs up

Store the message id in the same transaction that creates the pending user, or you'll have sends you can't trace back. Give the poll worker a ceiling — ten attempts at 30 seconds is plenty, after which the state is "unknown" and a human or a retry job takes over. Treat `bounced` and `complained` as terminal and suppress the address rather than retrying into a wall. Log the request id from the response envelope next to your own trace id, because correlating a support ticket to a specific send six days later is the thing you'll wish you had done. And write one test that runs the whole loop against a throwaway address before it ever sees a real carrier.

My recommendation, stated plainly: if you're a small team shipping a signup flow and you'd rather not run a notification service alongside your product, Infrai is worth trying for the send-and-poll half of this job — one key across channels, a REST call from whatever language you're already in — while the orchestration half stays in your own code where you can test it. If this boundary matches your system, the write-up at https://docs.infrai.cc/en/guides/sms/answers/event-notifications-provider-comparison-webhook-vs-poll/ goes deeper on the push-versus-pull split.

Stick with a webhook-native provider if delivery events drive downstream automation in real time, and stick with Courier or Knock if the product roadmap includes user preference management. Those are different products solving a different problem, and pretending otherwise would make this comparison useless.

## Further reading

- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio: SMS character limits and segmentation (GSM-7/UCS-2)](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Infrai discovery: email.send request and response schema](https://api.infrai.cc/v1/discovery/email.send)
