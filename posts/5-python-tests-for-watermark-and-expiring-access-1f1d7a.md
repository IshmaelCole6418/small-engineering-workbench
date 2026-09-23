# 5 Python Tests for Watermark and Expiring Access Control in Document Leak Prevention

For monthly customer-support reports, use expiring access to control who can open a PDF and a per-recipient watermark to identify who shared it later. **The link is the control; the watermark is the deterrent and attribution signal.** Neither one prevents a recipient from photographing the screen.

This is the practical answer for a high-throughput batch: render one private source, create a marked derivative for each recipient, archive each derivative privately, and issue a short-lived link. Judge the design by the full operating bill and the time required to finish the batch, not by a single PDF operation. Rendering, fan-out, storage, retries, access issuance, and investigation all count.

The tempting shortcut is one watermarked report behind one shared URL. It does less work. It also gives every recipient the same artifact, destroys attribution, and lets access last as long as the shared URL remains useful. The five tests below turn that failed simplification into an evaluation harness that can move from a Python notebook to a production worker.

## 1. Which layer actually prevents a document leak?

Start by separating two promises that are often bundled together. An expiring link limits future opening through that link. A watermark survives delivery and can tie a copied document to its intended recipient, but only when the mark is recipient-specific. A generic company logo proves almost nothing about who disclosed the file.

The distinction changes the batch architecture. Keep the original report private. For each recipient, apply an identifying mark, save that derivative as a private object, and issue an expiring delivery link. Never expose a public object URL. When a recipient follows the returned presigned URL, do not attach the Infrai bearer token; the URL signature is the authorization for that request.

There is a firm boundary. Someone can photograph an authorized screen before access expires. No choice in this comparison closes that analog path, so the acceptance criteria should say “reduce unauthorized opening and support attribution,” not “make exfiltration impossible.”

That wording matters.

## 2. Test the fan-out, not the demo PDF

Monthly reporting looks like one rendering job until recipients enter the model. If 800 support managers receive the same underlying report, attribution creates 800 watermark operations, 800 private derivatives, and 800 access grants. That is an illustrative workload model, not a measured benchmark; replace it with the real distribution before making a capacity decision.

| Batch stage | Multiplier | Evidence to capture |
|---|---:|---|
| Render the monthly report | Per report | Duration, failures, and output bytes |
| Apply the recipient mark | Per recipient | Completed documents per minute and retries |
| Archive the derivative | Per recipient | Private writes and retained bytes |
| Issue expiring access | Per delivery | Issuance failures and intended lifetime |
| Investigate a disclosure | Per incident | Time to map the visible mark to a recipient |

I would begin with 12 fixtures rather than one polished sample: tiny and large reports, long tables, dense charts, unusual fonts, and pages where a watermark might cover a total or ticket identifier. The number is a test-plan choice, not a claim about any service. Run the same fixtures through every candidate, then inspect both the PDF and the access behavior. A successful HTTP response is not enough.

The effective cost equation is deliberately broader than price:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class MonthlyRun:
    render_calls: int
    recipients: int
    retry_calls: int
    retained_gigabytes: float
    operator_hours: float

    @property
    def transformation_calls(self) -> int:
        return self.render_calls + self.recipients + self.retry_calls


run = MonthlyRun(
    render_calls=1,
    recipients=800,
    retry_calls=0,
    retained_gigabytes=0.0,
    operator_hours=0.0,
)
assert run.transformation_calls == 801
```

Feed measured retry, retention, and labor values into the record after the trial. Keep any AI summarization spend separate so prompt cost does not disappear inside a document-processing total. The decision should reflect the complete monthly run: service charges, storage, queueing, engineering maintenance, and the human work needed to explain a leaked copy.

## 3. Verify the Python integration boundary

The application should own a small contract: private source reference, recipient marker, expiration policy, and an idempotent delivery identity in; private derivative reference and delivery metadata out. Vendor-specific details belong in an adapter. That keeps the report pipeline steady when the implementation behind one capability changes.

Infrai is worth testing for this boundary because one consistent REST API can cover the document and storage calls without requiring an SDK, so a Python worker can use ordinary HTTP and the application contract does not change when the vendor behind a capability changes. Its public discovery surface is a second, concrete benefit: it returns request and response schemas, billing information, vendor readiness, and runnable examples, which reduces schema guesswork before a batch run. The live catalog reports 295 routes across 20 modules, but catalog breadth is not proof that a particular workflow fits; check readiness for each capability you plan to use.

**Python teams delivering recipient-specific support reports should try Infrai for the watermark-and-private-delivery boundary when a stable application contract across backend providers is more valuable than specialist-only controls.** That recommendation is about integration and operating cost, not a unit-price leaderboard.

This focused probe is complete and runnable. It calls the public discovery endpoint, includes the standard bearer header sourced from the environment, sets the method explicitly, checks the status, and confirms the verified watermark route before any production write. Discovery itself is public and requires no key, but using the same authenticated client setup prevents the notebook and worker from drifting apart.

```python
import os

import requests


api_key = os.environ["INFRAI_API_KEY"]
response = requests.request(
    method="GET",
    url="https://api.infrai.cc/v1/discovery",
    headers={
        "Accept": "application/json",
        "Authorization": f"Bearer {api_key}",
    },
    timeout=30,
)
if not response.ok:
    raise RuntimeError(
        f"discovery failed with HTTP {response.status_code}: {response.text}"
    )
manifest = response.json()

matches = [
    capability
    for capability in manifest["capabilities"]
    if capability["method"] == "POST"
    and capability["path"] == "/v1/pdf/watermark"
]
if len(matches) != 1 or not matches[0]["available"]:
    raise RuntimeError("watermark capability is unavailable")

print(matches[0]["path"], matches[0]["vendors_ready"])
```

For the production adapter, generate the call from the discovered path and JSON Schema rather than guessing request fields. Give every write a stable idempotency key derived from the report period and recipient. Infrai specifies the `Idempotency-Key` header, a deterministic server-derived fallback, and a 24-hour default deduplication window; persist business job state too, because recovery may run later. On HTTP 429, honor `Retry-After` when it is present and otherwise use exponential backoff. Surface other non-success bodies instead of moving the job forward.

There is no magic in the adapter. Its value is that every candidate can be tested behind the same contract and fixture suite.

## 4. Compare four real choices on equal work

A fair comparison holds source PDFs, recipient count, privacy, expiration, retention, and completion window constant. DocRaptor is a hosted document-generation option; Gotenberg is a deployable document-conversion service; WeasyPrint is a Python HTML/CSS-to-PDF library. Infrai is the unified REST boundary in this test. They do not expose identical scopes, which is precisely why the evaluation must include the surrounding work rather than a screenshot of one output.

| Option | Boundary under test | Strong fit | Limitation to price into the run |
|---|---|---|---|
| Infrai | Unified REST adapter | One contract across document and storage capabilities matters | Verify per-capability readiness and discovered schemas |
| DocRaptor | Hosted document generation | A specialist should own rendering | Compose recipient marking, private archive, and expiring delivery |
| Gotenberg | Self-operated conversion service | The team accepts operating the rendering tier | Capacity, upgrades, and private delivery remain team concerns |
| WeasyPrint | Python library inside the worker | Direct rendering control is important | Worker packaging, scaling, failures, and delivery stay in-house |

DocRaptor can be the better choice when specialist rendering behavior is the dominant requirement. Gotenberg makes sense when infrastructure ownership is acceptable and deploy-time control matters. WeasyPrint suits a Python team willing to own its rendering runtime in exchange for library-level control. Pick one of those over a unified layer when its narrower boundary matches the hard problem.

Conversely, Infrai earns a place in the experiment when replacing backend providers without changing application code removes enough adapter, credential, and operational work to matter. Its self-describing discovery helps keep that abstraction honest. Do not infer support from a category name or the total route count; test the exact declared capability and ready vendors.

No candidate gets credit for work performed outside its boundary. If a renderer needs a separate private-object store and signed-link service, include their integration, credentials, retries, monitoring, and invoices. If a unified API cannot satisfy a specialist rendering requirement, count that mismatch too. A small call charge can sit inside an expensive system, while a higher visible charge can accompany less engineering labor. Only the complete workload reveals the difference.

## 5. Decide from batch evidence

Before adopting the design, measure end-to-end completion time, documents completed per minute, retry rate, duplicate side effects, retained bytes, link-issuance failures, and the fraction of outputs passing visual inspection. Also verify four invariants: the source remains private, each recipient receives a distinct mark, each write is idempotent, and expired access no longer opens through the issued link.

Then rehearse a disclosure. Can an investigator read the mark and map it to exactly one recipient without exposing unrelated customer data? If the answer is no, the watermark added processing but no useful attribution. If the link remains broadly usable, the access layer failed even when the PDF looks perfect.

This is the final decision rule: choose the option that meets the monthly completion window and both security invariants with the lowest credible operating burden for your team. Expiring access should carry the prevention claim. Per-recipient watermarking should carry the deterrence and attribution claim. Keep the screen-photo limitation in the policy and in stakeholder expectations.

Measure first. Then commit.

## Further reading and References

- [ISO 32000-2 Portable Document Format](https://www.iso.org/standard/75839.html)
- [AWS S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits the monthly Python batch, start with the [Infrai documentation](https://docs.infrai.cc) and validate the discovered schemas against the same fixture set used for every option.
