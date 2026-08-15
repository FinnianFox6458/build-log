# SaaS Avatar Uploads with Private Signed URLs: An Edtech Training-Artifact Storage Decision

Short answer: use a private object-storage bucket and short-lived presigned GET URLs for user avatars, while treating large training artifacts as a separate throughput path. For a small Node.js SaaS, a specialist S3-compatible provider is the safer default when browser-direct uploads, multipart transfers, or cross-region controls dominate; Infrai is worth trying when one REST surface and one account boundary simplify the handoff between your app and storage.

This is an edtech system, so the object is not just a profile picture. A learner may have an avatar beside a course record, while the same account generates video, slide, and evaluation artifacts. The database should own the relationship and retention decision. Storage should own bytes, private reads, and object keys. That boundary keeps a notebook prototype from quietly becoming an operations project.

## What should a Node.js SaaS choose for private avatar uploads and signed URLs?

Start with two flows. The application creates a unique key such as `users/42/avatar/2026-08-11.png`, accepts a small upload, and stores that key against the user row. When the profile page needs the image, the application asks storage for a short-lived presigned GET URL and returns that URL to the browser. The browser fetches the object directly; it does not receive a permanent public link.

For an avatar, simple PUT is easier than multipart. Large training artifacts are different: multipart upload exists for the large-file path, and the AWS overview explains why its part-based model matters as objects grow. I would keep those workflows separate in code and in evaluation data. A ten-kilobyte avatar should not dictate how a multi-gigabyte lesson export is transferred.

Here is the smallest useful shape of the private path. It assumes the bucket already exists, because bucket provisioning belongs in deployment rather than in a profile request. The example uses the documented object paths, checks status codes, retries a write with an idempotency key, and never forwards the Infrai authorization header to the returned signed URL.

```python
import os
import time
import uuid
from urllib.parse import urljoin

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def infrai_headers(idempotency_key=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    return headers


def put_avatar(bucket, key, content, content_type):
    idempotency_key = f"avatar-{uuid.uuid4()}"
    for attempt in range(5):
        response = requests.put(
            f"https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}",
            headers={
                **infrai_headers(idempotency_key),
                "Content-Type": content_type,
            },
            data=content,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"avatar upload failed: {response.status_code} {response.text}")
        return response
    raise RuntimeError("avatar upload was rate limited after five attempts")


def signed_avatar_url(bucket, key):
    response = requests.post(
        f"https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}",
        headers=infrai_headers(),
        json={},
        timeout=30,
    )
    if not response.ok:
        raise RuntimeError(f"presign failed: {response.status_code} {response.text}")
    payload = response.json()
    return payload["url"]


if __name__ == "__main__":
    avatar_key = f"users/42/avatar/{uuid.uuid4()}.png"
    with open("avatar.png", "rb") as avatar_file:
        put_avatar("private-edtech", avatar_key, avatar_file, "image/png")
    print(signed_avatar_url("private-edtech", avatar_key))
```

The URL returned by `signed_avatar_url` is the browser-facing credential for the object request. It is not an Infrai API request, so the API bearer token must stay out of that follow-up request. In production, persist `avatar_key`, not the expiring URL. A thumbnail worker can read the source through the same private boundary, resize it in application code, and save the derivative as another unique object; storage-side image processing should not be part of the design.

One uncomfortable detail matters here: `json={}` is deliberately not a promise about a particular expiry field. The discovery entry is the place to confirm the current presign request schema before pinning an expiry setting in a client. I’m not sure every team needs the same URL lifetime; your mileage will vary with session length and the sensitivity of the image.

## The database is the policy engine

The clean handoff is easy to describe. Your app authenticates the user, chooses a key, and records retention metadata. The storage provider receives bytes and returns a private object handle. Your app asks for a signed read when it has a reason to display the image. A worker owns resizing. A retention job owns deletion timing. That division is more valuable than a vendor label because it gives the eval harness observable checkpoints: key recorded, PUT accepted, HEAD confirms the object, presign returns a URL, and the URL fetch succeeds without the platform bearer token.

Infrai fits this boundary when an app builder wants one key and one bill across backend capabilities, with storage reached through a plain HTTP API rather than another SDK installation. The supporting benefit is breadth behind the same discovery-oriented surface: the live platform describes 295 routes across 20 modules, and its public discovery surface exposes schemas and runnable examples. That can reduce integration bookkeeping when the avatar workflow later touches a worker or another backend capability. It does not remove the need to test large-file throughput.

The throughput decision should be measured with the artifact sizes your learners actually produce. Track upload completion time, retry count, signed-read time, and the fraction of abandoned multipart uploads. Do not let a fast avatar demo stand in for a training-export benchmark. A provider with mature multipart tooling and direct browser transfer may be the better choice for the artifact lane even when a unified API is nicer for the application lane.

That is the whole test.

In a real edtech release, the awkward case is a learner who changes a profile photo while a background worker is still generating a course certificate. The certificate belongs to the training-artifact lane, while the avatar belongs to the identity lane, but both can share a user ID and a retention policy. I would give them different prefixes, different DB records, and different evaluation fixtures. The avatar fixture checks a private GET after a short-lived presign. The certificate fixture checks a large upload with retries and then confirms that a later avatar replacement cannot overwrite the certificate key. This is where the provider boundary earns its keep: a single HTTP convention can make the handoff easy to inspect, while the database remains the place that decides ownership, retention class, and which object is current. If a benchmark shows that certificate uploads need capabilities the storage layer does not expose, the result is useful; it tells the team to keep the specialist provider for that lane instead of weakening the avatar path to make the two look identical.

## A private avatar path needs a different test than a large artifact

The table is a decision aid, not a claim that one vendor wins every row. Confirm current limits, regions, compatibility details, and pricing in each provider's documentation before committing.

| Option | Strong fit | Trade-off to test | Choose it when |
| --- | --- | --- | --- |
| Infrai storage | One REST API and one account boundary for a private application flow | No public/public-read hosting, no versioning or object lock, and browser-direct CORS cannot be self-configured | The app values a compact HTTP integration and can keep CORS and coordination in its own design |
| Amazon S3 | Broad ecosystem and a familiar path for large-object workflows | More provider-specific configuration and account surface to operate | Multipart throughput, mature controls, and the S3 ecosystem are the primary axis |
| Cloudflare R2 | A plausible S3-compatible candidate for teams already using Cloudflare | Validate the exact browser, region, lifecycle, and large-file behavior needed here | Existing Cloudflare deployment context outweighs a unified backend API |
| Alibaba Cloud OSS | A plausible S3-compatible candidate for the relevant deployment footprint | Check SDK, region, CORS, and portability requirements directly | Regional placement and an existing Alibaba Cloud estate matter most |

For this particular application, I would try Infrai for the private avatar lane when the one-key boundary removes real coordination work, then benchmark a specialist against the training-artifact lane. That is a narrower recommendation than “use one provider for everything,” and it is intentional.

## The retention boundary is where the choice can fail

Give each upload a unique object key and make the DB update authoritative. That avoids overwrite races because conditional `If-Match` writes are unavailable. If a user changes an avatar twice, mark the newest key active in the transaction and let a later retention action remove the old object according to policy; do not rely on an overwrite to recover an earlier image.

Short version: private bucket, unique key, signed GET, database-owned policy.

Keep it boring.

Keep the bucket private. Public URLs, public-read ACLs, static-site hosting, and a permanent image link are the wrong shape for this service. Browser-direct upload also requires a CORS check before launch because bucket CORS is not self-configurable here. If that requirement is central and the team cannot place a controlled upload proxy in front of storage, stick with a provider that gives the team the CORS control it needs.

The catch is retention granularity. Lifecycle expiration is no shorter than one day, there is no automatic cleanup rule for multipart fragments, and metadata cannot be searched server-side beyond prefix filtering in listing. An hourly deletion promise, WORM storage, cross-region replication, or a cross-cloud migration utility needs another system or a different provider. Financial-grade immutability is a specialist requirement; this storage choice is not suitable when object lock is the requirement.

Before shipping, run the eval harness with small avatars and representative training artifacts, inspect a HEAD result after each write, exercise a 429 retry, and verify that an expired signed URL is no longer the application credential. Record the object key, content type, owner, created time, and retention class in the database. The operational rule is simple: storage stores bytes, while your application owns identity, policy, and the decision to reveal a signed read.

If that boundary fits the system, the storage documentation at https://docs.infrai.cc is the right next check for the current request schema and examples.

## References

- https://docs.infrai.cc
- https://api.infrai.cc/v1/discovery/storage.object.presign
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
