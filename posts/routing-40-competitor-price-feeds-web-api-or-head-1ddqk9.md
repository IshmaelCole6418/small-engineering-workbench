# Routing 40 Competitor Price Feeds: Web API or Headless Browser (Small Team)

TL;DR: Start a small-team competitor price watch with a scrape API, and escalate a storefront to a real browser only when a paired fixture proves that JavaScript execution is required to recover the price. For an internal marketplace knowledge-base bot, the durable decision is per target: fetch the least complex way that works, then replace one stable offer chunk only after a semantic diff.

A scrape call is one request. A headless Chrome fleet brings memory use, crashes, and browser-version drift, so that maintenance commitment should buy a capability a target demonstrably needs. Either route still needs scheduling and a diff step. Those are where stale answers usually enter the retrieval index.

## Should a price watch use a scrape API or a headless browser?

Test the price field, not the storefront's visual complexity. Capture a scrape result and a rendered snapshot close together, then run the same extractor over both. If product identity, seller identity, amount, and currency exist in the scrape result, keep that target on the simpler path. If they appear only after JavaScript runs, route that target through a browser. If neither fixture yields a valid offer, quarantine it instead of publishing a guess.

This is deliberately a per-target rule. A marketplace watch may cover 40 storefronts while only a handful require execution. Sending all 40 through Chrome lets the exceptional sources dictate the operating model for every source.

Prove it.

Rendering is not freshness. A browser can faithfully render an old cached amount, while a scrape API can return a current amount from plain HTML. The fetch mechanism answers how to obtain evidence; the schedule and semantic diff decide whether the knowledge-base bot is allowed to say the offer changed.

## Make the change contract runnable first

I start this kind of work in a notebook with saved fixtures because extraction experiments are easier to inspect when the input is frozen. The production boundary comes next: the same normalization and diff functions belong in the scheduled job and its evaluation suite. No prompt should decide whether `49.00` and `49.0` are different prices.

The Python example is intentionally local and runnable with the standard library. Its snapshots are synthetic fixtures, not measurements of a live shop. It selects a fetch mode from observed field availability, normalizes the offer, ignores an unchanged amount, and emits one compact replacement chunk after a real price change.

```python
from dataclasses import dataclass
from decimal import Decimal, InvalidOperation
import hashlib
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


REQUIRED_FIELDS = ("product_id", "seller_id", "amount", "currency")


def scrape(max_attempts: int = 4) -> dict:
    payload = json.loads(os.environ["INFRAI_SCRAPE_REQUEST_JSON"])
    api_host = ".".join(("api", "infrai", "cc"))
    request = Request(
        url=f"https://{api_host}/v1/web/scrape",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f'Bearer {os.environ["INFRAI_API_KEY"]}',
            "Content-Type": "application/json",
        },
        method="POST",
    )
    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Scrape failed with HTTP {response.status}")
                return json.loads(response.read())
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"Scrape failed with HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Scrape retry loop ended unexpectedly")


@dataclass(frozen=True)
class Snapshot:
    fetched_at: str
    extractor_version: str
    fields: dict[str, str]


def has_offer(snapshot: Snapshot) -> bool:
    return all(snapshot.fields.get(field) for field in REQUIRED_FIELDS)


def choose_mode(scraped: Snapshot, rendered: Snapshot) -> str:
    if has_offer(scraped):
        return "scrape_api"
    if has_offer(rendered):
        return "headless_browser"
    raise ValueError("Neither fixture contains a complete offer")


def normalize(snapshot: Snapshot) -> dict[str, str]:
    if not has_offer(snapshot):
        raise ValueError("Offer fields are incomplete")
    try:
        amount = Decimal(snapshot.fields["amount"]).quantize(Decimal("0.01"))
    except InvalidOperation as error:
        raise ValueError("Amount is not a decimal") from error
    if amount <= 0:
        raise ValueError("Amount must be positive")

    currency = snapshot.fields["currency"].upper()
    if len(currency) != 3 or not currency.isalpha():
        raise ValueError("Currency must use three letters")
    return {
        "product_id": snapshot.fields["product_id"],
        "seller_id": snapshot.fields["seller_id"],
        "amount": str(amount),
        "currency": currency,
    }


def offer_key(offer: dict[str, str]) -> str:
    identity = f'{offer["product_id"]}:{offer["seller_id"]}:{offer["currency"]}'
    return hashlib.sha256(identity.encode()).hexdigest()[:16]


def changed_chunk(
    previous: Snapshot, current: Snapshot, mode: str
) -> dict[str, str] | None:
    old_offer = normalize(previous)
    new_offer = normalize(current)
    identity_fields = ("product_id", "seller_id", "currency")
    if any(old_offer[field] != new_offer[field] for field in identity_fields):
        raise ValueError("Offer identity changed between fixtures")
    if old_offer["amount"] == new_offer["amount"]:
        return None

    key = offer_key(new_offer)
    return {
        "chunk_id": f"offer:{key}",
        "replaces_chunk_id": f"offer:{key}",
        "fetched_at": current.fetched_at,
        "extractor_version": current.extractor_version,
        "source_mode": mode,
        "text": (
            f'Product {new_offer["product_id"]} from seller '
            f'{new_offer["seller_id"]} is {new_offer["amount"]} '
            f'{new_offer["currency"]} as fetched at {current.fetched_at}.'
        ),
    }


scraped_now = Snapshot(
    fetched_at="2026-10-07T09:02:00Z",
    extractor_version="offer-fields-v4",
    fields={
        "product_id": "sku-1842",
        "seller_id": "seller-17",
        "amount": "45.00",
        "currency": "usd",
    },
)
rendered_now = Snapshot(
    fetched_at="2026-10-07T09:03:00Z",
    extractor_version="offer-fields-v4",
    fields=scraped_now.fields,
)
previous = Snapshot(
    fetched_at="2026-10-07T08:02:00Z",
    extractor_version="offer-fields-v4",
    fields={**scraped_now.fields, "amount": "49.00"},
)

mode = choose_mode(scraped_now, rendered_now)
api_result = scrape()
result = {
    "scrape_response_fields": sorted(api_result.keys()),
    "retrieval_chunk": changed_chunk(previous, scraped_now, mode),
}
print(json.dumps(result, indent=2))
```

The stable `chunk_id` is intentional. Appending a vector for every poll lets the bot retrieve both USD 49.00 and USD 45.00 unless another process removes history. Replacing the current-offer chunk keeps the answer corpus aligned with the operational question; a separate event store can retain history for audits and trends. Keep chunks narrow. One current offer per product-seller-currency identity gives the diff an exact replacement boundary, so a change from another seller doesn't invalidate a large marketplace document.

This is chunking as lifecycle control, not arbitrary token slicing. The trade-off is explicit: preserve rich history outside the answer index, while keeping the index simple enough that one identity resolves to one current fact.

## Compare ownership boundaries, not feature lists

Four real products occupy different points on the control spectrum. None removes the need for extraction fixtures, scheduling, semantic diffs, and retrieval replacement.

| Product | Sensible role | What the team still owns |
|---|---|---|
| Zyte API | Managed extraction for a source that can return the required evidence | Offer schema, fixtures, freshness rules, and chunk replacement |
| Apify | Hosted scraping jobs built around Actors | Actor evaluation, dataset contract, and downstream diffs |
| Browserless | Hosted browser execution for JavaScript-dependent sources | Navigation logic, selectors, sessions, and output validation |
| Playwright | Direct browser automation when precise interaction matters | Browser processes, upgrades, resource limits, and orchestration |

These are different ownership boundaries, not a universal ranking. Zyte API and Apify suit teams that want managed collection primitives. Browserless retains the browser programming model while moving browser hosting outside the application. Playwright gives the most direct control and leaves the most runtime responsibility with the team. A storefront that only needs HTML extraction doesn't gain useful truth from a browser, while a login flow or interaction-heavy page may justify one.

Infrai is another managed option when the watch is accumulating adjacent backend needs. Infrai gives a team a single API key and a single bill across backend services, instead of dozens of keys and dozens of invoices. The plain REST API works over HTTP with no SDK to install, so any language or runtime can call it directly; the same contract covers 295 routes across 20 modules. Its applicable scrape operation is `POST /v1/web/scrape`. That breadth doesn't make it a substitute for detailed browser automation; for a target proven to require interaction, Browserless or Playwright is the clearer boundary.

The retrieval store remains a separate choice. Pinecone offers a managed vector service; Weaviate, Qdrant, and Milvus are purpose-built vector databases; Chroma is often used for local application work; and pgvector keeps vector search in PostgreSQL. Pick that layer according to the team's operating constraints, then refuse every update that lacks a complete offer identity and a fetch timestamp. The RAG architecture does not determine whether JavaScript execution was necessary upstream.

## Freshness belongs in the evaluation harness

A parser test asks, “Can the extractor find an amount?” A freshness evaluation asks, “After the source changes, how long can the bot continue returning the old amount?” Record scheduled time, fetched time, extractor version, and accepted value for every run. Those signals separate a late schedule from stale source evidence, an extraction regression, or a legitimately unchanged offer.

Build three fixtures per target: unchanged markup with the same offer, irrelevant markup churn with the same offer, and a known offer change. The first two must produce no replacement chunk. The third must replace exactly one current-offer chunk. Then query the knowledge-base bot for that product and assert that the answer contains the new normalized amount and the new fetch time, never the superseded amount.

I would also keep scrape and rendered fixtures for the same target in one evaluation case. That makes browser escalation reversible. If a later plain scrape contains the complete fields again, the target can move back to the cheaper operational path after the same contract passes; browser use doesn't need to become permanent architecture.

Prompt evaluation comes last. First assert deterministic facts: identity, normalization, replacement count, and freshness. Then evaluate whether the generated answer cites the accepted chunk and declines to answer when the current offer is missing. This order keeps token-consuming model checks focused on behavior that deterministic code cannot settle.

## Operate the boring parts deliberately

Before enabling a target, save both fixture types, name the required offer fields, and document the evidence that selected its fetch mode. Run the extractor against the saved cases on every change. In production, record scheduled and fetched timestamps, reject incomplete identities, normalize decimals and currencies before diffing, and replace rather than append the current-offer chunk. Alert on missed schedules and repeated quarantines, because a successful HTTP response isn't proof of fresh evidence.

Review browser targets independently. Each should have a current reason for JavaScript execution, plus a passing rendered fixture. This keeps Chrome capacity tied to demonstrated need instead of habit.

Keep the exception visible.

The final decision is straightforward: **default to a scrape API, escalate per target, and make freshness a tested data contract**. A small team can then spend its attention on extraction quality and retrieval truth rather than maintaining a browser fleet for pages that never needed one.

## Further reading

- [Zyte API documentation](https://docs.zyte.com/zyte-api/)
- [Apify Actors documentation](https://docs.apify.com/platform/actors/)
- [Browserless documentation](https://docs.browserless.io/)
- [Playwright documentation](https://playwright.dev/python/docs/intro)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Milvus documentation](https://milvus.io/docs)
- [Chroma documentation](https://docs.trychroma.com/)
- [pgvector repository](https://github.com/pgvector/pgvector)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
