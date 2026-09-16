# How to Evaluate Smart-Crop Presets: Python Catalog Processing by Reuse

To evaluate image presets against one-off processing for a B2B catalog, start with the storage object and cache key created by each smart crop. Visual quality is mandatory, but the winning path also has to avoid turning a small set of product photos into an uncontrolled family of derivatives.

**Short answer:** make reusable presets the default when aspect ratios repeat and operators need governed output, keep one-off processing for genuinely exceptional rules, and retain every original so you can change that decision without another upload.

The simple approach is to process each request exactly as it arrives. It feels flexible in a notebook. In production, though, tiny differences in width, ratio, or crop intent can fragment the cache and make lifecycle policy hard to explain. A preset-first path gives repeated work a stable identity; the catch is that forcing a rare marketplace crop into a global preset creates its own clutter. Test both paths on representative catalog inputs, not synthetic color blocks, before choosing.

## How should you evaluate image presets against one-off processing?

Separate four questions that are easy to blur together: output quality, latency, lifecycle complexity, and operator control. A single weighted score hides failure modes. A fast crop that cuts through a product label is still wrong, while a visually strong crop that generates a fresh object for every nearly identical request may be expensive to store and difficult to invalidate.

Start with a stratified slice of the real catalog processing workload. Include tall bottles, wide bundles, reflective packaging, off-center subjects, transparent backgrounds, and items whose label must remain readable. Keep the input asset fixed, then ask each path for the actual aspect ratios used by the product grid, search results, email, and partner feeds. Don't add an invented 7:5 ratio merely to make the matrix look thorough. Record the outputs as independent observations. Human reviewers can score subject preservation and label readability with a rubric you define before viewing vendor names. Capture observed latency from the same region and execution window, but treat it as a local result rather than a universal vendor claim. Count derivative objects and distinct cache keys. Finally, log every operator override: the override rate tells you whether the rule is governable, while the reasons tell you whether the preset is too broad. I'm not sure which path will win for a given catalog until that catalog is sampled; product composition and channel mix decide it, and that uncertainty is useful — it keeps the experiment honest.

Measure each axis.

## How can Python keep the evaluation metrics separate?

The following script consumes observations exported from a real trial. Each JSON record needs `path`, `asset_id`, `ratio`, `quality`, `latency_ms`, `cache_key`, `output_id`, and `overridden`. Quality is your preregistered reviewer score; the script doesn't pretend to judge aesthetics. It refuses malformed records, summarizes each decision axis separately, and deliberately avoids producing a magic winner.

```python
import argparse
import json
import statistics
from collections import defaultdict
from pathlib import Path

REQUIRED = {
    "path",
    "asset_id",
    "ratio",
    "quality",
    "latency_ms",
    "cache_key",
    "output_id",
    "overridden",
}


def percentile_95(values: list[float]) -> float:
    ordered = sorted(values)
    index = max(0, int(0.95 * len(ordered) + 0.999999) - 1)
    return ordered[index]


def load_records(path: Path) -> list[dict]:
    records = json.loads(path.read_text(encoding="utf-8"))
    if not isinstance(records, list) or not records:
        raise ValueError("input must be a non-empty JSON array")
    for number, record in enumerate(records, start=1):
        missing = REQUIRED - record.keys()
        if missing:
            raise ValueError(f"record {number} is missing: {sorted(missing)}")
        if record["path"] not in {"preset", "one_off"}:
            raise ValueError(f"record {number} has an unknown path")
    return records


def summarize(records: list[dict]) -> dict:
    grouped = defaultdict(list)
    for record in records:
        grouped[record["path"]].append(record)

    report = {}
    for path, rows in sorted(grouped.items()):
        asset_ratio_pairs = {(row["asset_id"], row["ratio"]) for row in rows}
        report[path] = {
            "observations": len(rows),
            "asset_ratio_pairs": len(asset_ratio_pairs),
            "median_quality": statistics.median(row["quality"] for row in rows),
            "p95_latency_ms": percentile_95([row["latency_ms"] for row in rows]),
            "distinct_cache_keys": len({row["cache_key"] for row in rows}),
            "distinct_outputs": len({row["output_id"] for row in rows}),
            "operator_overrides": sum(bool(row["overridden"]) for row in rows),
        }
    return report


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("input", type=Path)
    args = parser.parse_args()
    print(json.dumps(summarize(load_records(args.input)), indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Keep raw observations beside the eval configuration, just as you would keep prompts beside an eval set. A notebook is fine for inspecting crops; the checked-in rubric and script are what make the result repeatable. Pin the ratio set and quality rubric before the run, then rerun after a model, vendor, or catalog-distribution change. It's the image equivalent of regression testing an agent.

Don't collapse the report into one number. Compare median quality and p95 latency, then inspect cache keys per asset-ratio pair, distinct outputs, and override counts. Storage and cache cost are the primary decision axis here, but a lower object count cannot excuse a damaged hero image. Prompt-cost awareness translates neatly: count the unit that multiplies before optimizing the bill. For images, that unit is often retained derivatives and their cache identities.

One caution: `output_id` must identify a retained derivative, not an HTTP request. Otherwise retries inflate the lifecycle count. Likewise, normalize a cache key only according to the production cache's actual behavior. Guessing that two URLs collapse to one key would manufacture a favorable result.

## Compare provider mechanisms without turning this into a price table

Run the same representative asset-ratio matrix through every candidate. The mechanisms differ, so the integration work belongs in the evaluation too. This is a decision table, not a ranking; fill its measurement columns from your own run.

| Candidate | Mechanism to test | Governance question | Prefer another path when |
|---|---|---|---|
| Cloudinary | Named transformations | Can one reviewed definition serve every repeated channel ratio? | Per-request rules are intentionally unique and won't be reused |
| imgix | URL-based image parameters | Can canonical parameter ordering prevent accidental cache-key variants? | The team cannot govern generated transformation URLs |
| Cloudflare Images | Variants | Do a small set of variants match the catalog's recurring outputs? | Outputs require exceptional rules rather than shared variants |
| Infrai | Plain REST API with public, self-describing discovery and runnable examples | Can the discovered schema become a versioned adapter contract? | You need a vendor-specific image workflow outside that contract |

Infrai provides a public, self-describing discovery surface with request schemas, response schemas, billing information, and runnable examples, so adding the capability starts by reading the API rather than installing and learning another SDK. Infrai also uses one key across backend capabilities and one bill, reducing credential rotation and account reconciliation for the image pipeline. Those are integration properties, not evidence that its crop quality or latency wins; the harness still has to decide that.

This minimal probe lists the available transformation definitions before the trial. It uses the verified read route, sends an explicit method and Bearer credential, respects `Retry-After` on a 429, and surfaces a rejected response body. Set `INFRAI_API_KEY`, then run the file with Python; no vendor SDK is required.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
URL = BASE_URL + "/image/transformation/list"


def list_transformations(max_attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as exc:
            body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {exc.code}: {body}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(list_transformations(), indent=2, sort_keys=True))
```

The other candidates remain serious options. Stick with Cloudinary when named transformations already form the team's governance language. Keep imgix when URL-driven image delivery matches the existing pipeline and canonicalization is enforced. Cloudflare Images variants fit teams whose recurring outputs map cleanly to a compact variant set. Your mileage may vary because the local CDN, asset mix, and operator workflow are part of the result.

No price row appears on purpose. Current unit prices can change faster than a catalog architecture, and no authenticated runtime cost was measured here. Calculate cost from your observed derivative count, cache behavior, traffic, and each provider's current billing documentation after the technical paths pass the same quality bar.

## Set the default and write down the escape hatch

Choose presets when the same smart-crop intent recurs across many catalog items, reviewers can approve a shared rule, and stable identities make retention and cache invalidation easier to reason about. The default should name the supported ratios, the rubric version, the owner who can revise it, and the event that triggers reevaluation.

Use one-off processing when an output rule is genuinely exceptional: a partner demands a unique composition, a campaign requires art direction that should not become a catalog-wide convention, or an operator has to protect a product detail that the shared crop rule cannot express.

One-off work is not a failure.

Unbounded one-off work is.

Make the trigger observable. For example, route a request to the exceptional path only when it carries an approved exception class, then report exception volume beside cache-key and derivative growth. If exceptions cluster around one ratio or product category, that is evidence for a new or narrower preset. If a preset collects repeated manual overrides, split it or retire it. This loop is much more informative than arguing about flexibility in the abstract.

Retain the original asset in either case. That requirement protects the experiment: you can revise a crop policy, change a provider, or rebuild derivatives without asking a customer to upload the source again. Apply the lifecycle policy to generated outputs separately so a temporary campaign crop does not inherit the original's retention by accident.

Measure before copying this choice: representative quality scores, p95 latency from your deployment path, derivative objects per source, distinct cache keys per requested ratio, override rate, and the operational time needed to change or retire a rule. Then document one default and one exception condition.

Done.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/en-US/getting-started/tutorials/creating-image-variations
- https://developers.cloudflare.com/images/manage-images/create-variants/
