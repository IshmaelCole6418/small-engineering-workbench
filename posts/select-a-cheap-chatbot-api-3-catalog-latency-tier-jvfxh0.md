# Select a Cheap Chatbot API: 3 Catalog Latency Tiers for Quality Tradeoffs

Short answer: the cheapest chatbot API is the one that meets a defined catalog-quality floor with the fewest retries, escalations, and wasted tokens. This is my how-to for making that selection: use a small labeled set, three latency tiers, and a routing rule that measures field accuracy separately from response speed. A low per-request price is a weak decision signal if a bad package dimension or material label reaches search, quoting, or warehouse workflows.

The practical flow is compact: normalize the messy description, ask for a constrained catalog record, validate that record, and route uncertain cases to a slower or more capable path. I keep the interface provider-neutral so the evaluation harness can swap an API without rewriting the enrichment pipeline. The interesting work is deciding what quality means before anyone argues about price. The tradeoff is visible in the data, not in a slogan.

That is the trap.

## How do I select a chatbot API without trading away catalog quality?

A catalog description such as `12pk nitrile gloves, med, blue, powder free` contains several fields, but it does not promise a stable format. Another listing may put units first, omit the material, or use an abbreviation that only a domain specialist recognizes. The model has to extract what is present, leave what is absent unknown, and avoid turning a guess into a fact.

I score the output field by field. Exact agreement works for identifiers and counts. Normalized agreement works for units, casing, and common spelling variants. A human review label is appropriate for safety-relevant or ambiguous attributes. One aggregate score hides the failure that matters: a record can look good overall while consistently inventing dimensions.

For an in-app chatbot, latency also has more than one meaning. A synchronous catalog lookup needs a response budget that feels immediate to the person asking about an item. A nightly enrichment job can accept a queue and a later result. Those are different products, even if they call the same endpoint. I would rather route a small ambiguous slice to a slower path than make every request pay that latency.

The decision rule I use is a constraint, not a ranking: choose the lowest-cost tier whose validated field accuracy clears the business floor, then measure p95 latency and retry rate. If the floor is missed, price cannot rescue the design.

Here is the compact analysis I carry into a review:

| Tier | Best fit | Main risk | Gate |
| --- | --- | --- | --- |
| Fast | Clear, short descriptions | Missed nuance | Field checks pass |
| Standard | Ordinary catalog text | More latency | Quality floor passes |
| Review | Ambiguous or failed records | Human queue grows | Evidence is inspectable |

The table is a routing policy, not a product ranking.

## A runnable Python baseline for constrained enrichment

The example below keeps transport behind a tiny interface. The endpoint is intentionally pseudonymous; the important parts are the schema, the validation boundary, and the evidence passed to the next decision. It uses only Python's standard library.

```python
from dataclasses import dataclass
from typing import Any
import json


@dataclass
class EnrichmentResult:
    record: dict[str, Any]
    confidence: float
    needs_review: bool


def build_prompt(description: str) -> str:
    return f"""Extract a logistics catalog record from this description.
Return JSON with exactly these keys: name, quantity, unit, material, dimensions, evidence.
Use null for an attribute that is not stated. Do not infer missing values.
Description: {description}"""


def validate_record(payload: dict[str, Any]) -> None:
    required = {"name", "quantity", "unit", "material",
                "dimensions", "evidence"}
    if set(payload) != required:
        raise ValueError("schema mismatch")
    if not isinstance(payload["evidence"], list):
        raise ValueError("evidence must be a list")
    if payload["quantity"] is not None and not isinstance(
        payload["quantity"], int
    ):
        raise ValueError("quantity must be an integer or null")


def enrich(description: str, call_model) -> EnrichmentResult:
    raw = call_model(build_prompt(description))
    payload = json.loads(raw)
    validate_record(payload)

    stated = len(payload["evidence"])
    confidence = min(1.0, stated / 3.0)
    needs_review = confidence < 1.0
    return EnrichmentResult(payload, confidence, needs_review)
```

The `call_model` function is the only provider-specific seam. In production I would add a timeout, a request identifier, and bounded retries around it, then record the original description, parsed fields, validation result, latency, and token counts under the same identifier. I would redact customer or supplier data before sending traces to a shared system.

The prompt asks for evidence because a value without a source span is hard to review. Evidence is not proof of correctness, so the evaluator still compares the output with a labeled answer. It is a review aid and a useful signal for routing.

## How should the quality and latency tiers be tested?

Build a fixed evaluation set before choosing a runtime. I would include ordinary descriptions, abbreviations, missing fields, conflicting quantities, long descriptions, and adversarial text that tries to change the output format. Split the set by failure mode, not just at random. A random split can make a system look healthy while hiding the one category that drives operational damage. The analysis should name that category, the affected field, and the action taken when it appears.

Run each candidate tier against the same inputs and capture at least four measures: field-level accuracy, invalid-schema rate, p95 latency, and token usage. Add a review rate if humans are in the loop. The test harness should preserve raw outputs and evaluator decisions so a prompt change can be compared with its predecessor.

A useful routing table has three rows, even if the implementation starts with two. The fast tier handles clear, short descriptions. The standard tier handles normal catalog text. The review tier handles missing evidence, schema failures, conflicting attributes, or low confidence. These labels describe behavior, not products.

Do not use one confidence number as a substitute for evaluation. A model can be confidently wrong. In my Python RAG and agent work, I initially treated confidence as a convenient decision boundary; I later found that it is only a routing hint until calibrated against labeled examples. The calibration set should be refreshed when suppliers, categories, or language patterns change.

Token cost deserves its own test. A verbose prompt repeated for every row can dominate the economics even when the API's unit price looks favorable. Keep instructions stable, put the changing description in one clearly delimited region, and avoid returning prose that the application will discard. The cheapest useful output is the smallest output that preserves validation and reviewability.

This method has a boundary. It is a poor fit when a catalog field cannot be judged from text alone; that field needs a domain workflow, an image, or a human decision. A cheaper API does not remove that missing evidence.

## Where do retries, idempotency, and fallbacks change the result?

A failed request is not automatically safe to repeat. The application needs to know whether the operation is read-only, whether a partial result was stored, and whether repeating it could create a duplicate downstream action. RFC 9110 distinguishes idempotent methods and explains why a client may retry an idempotent request after a communication failure; that transport rule does not make a model call semantically idempotent by itself.

Give each enrichment job a stable idempotency key derived from the catalog item revision. Store the request state before dispatch, accept a result only after schema validation, and make the write conditional on that revision still being current. A retry can then re-evaluate the same revision without overwriting a newer supplier update.

The fallback path should preserve the same output contract. If a fast call times out, the queue can move the item to the standard tier. If parsing fails twice, the item can go to review with the original text and the validation error. Do not silently substitute an empty field: `null` with evidence about absence is different from a missing key and different again from an invented value.

Record it.

One operational trap is measuring only successful calls. Track timeouts, invalid JSON, empty evidence, retry count, review volume, and stale-result rejects. These are the events that turn a nominally cheap integration into an expensive operations queue.

## A decision checklist that survives a model change

Before selecting a tier, write down the minimum acceptable accuracy for each field and the fields that require human review. Keep latency targets separate for interactive chat and asynchronous enrichment. Run the same labeled fixtures through every candidate, and publish the failure categories beside the aggregate score.

Then calculate total work per accepted record: initial tokens, retries, validation failures, human reviews, and reprocessing after catalog revisions. Compare that number with the value of the resulting record, not with a headline request price. A cheaper tier that sends twice as many records to review is not cheaper for the workflow.

Finally, make the decision reversible. Keep the provider call behind the interface shown above, version prompts and schemas, and retain enough input and output metadata to reproduce a decision without retaining more sensitive text than policy permits. Re-run the fixture set after changing a prompt, a parser, a category taxonomy, or a routing threshold.

That is the selection method: establish a quality floor, separate synchronous latency from batch latency, measure total work, and choose the lowest tier that passes. The answer can change as catalog language changes. The evaluation harness is what makes that change visible.

## Further reading

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
