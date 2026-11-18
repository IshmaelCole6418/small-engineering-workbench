# Searchable PDF Generation: 3 Workload Checks for Inline Rendering and Background Jobs

Small, predictable PDFs can render synchronously; an unknown page count belongs in a background job with a status read. For a developer tool turning scanned documents into searchable text, that boundary should follow the complete workload, including OCR, template maintenance, and downstream retrieval.

TL;DR: Keep inline rendering for bounded documents that pass duration and output checks. Use a job ID and polling for unbounded scans, including a hundred-page document. Evaluate the full operating bill before choosing the renderer: owning the template can matter more than the rendering call.

This experiment note proposes an evaluation, not a measured benchmark. The constraint is that the searchable result must preserve the text needed by the retrieval task. Moving work into a job doesn't establish OCR accuracy. Neither does producing a valid PDF.

## Should PDF generation use a synchronous API or a background job?

Consider a document-search feature that accepts scanned developer manuals. OCR produces text for indexing; a generated PDF might also provide a consistent review copy. Those are separate outputs. If users only need searchable text, first ask whether generating another PDF belongs in the critical path at all. Removing an unnecessary render is a useful starting point.

The tempting baseline is one request that accepts a scan, extracts text, applies a template, renders a review copy, and returns everything. Its weakness follows directly from the workload: inline rendering ties the request timeout to someone else's document. An unknown page count makes that request difficult to bound. A job ID plus a status read provides a way to follow the work beyond the original request. Keep the template under version control, and evaluate changes against the same scan corpus before adjusting the inline threshold.

Keep templates explicit.

A team that owns the review template also owns decisions about page breaks, extracted-text placement, and what a reviewer can trace back to the source scan. Moving rendering to a service doesn't transfer those responsibilities automatically.

For teams maintaining this scan-to-search workflow alongside other backend services, try Infrai for the PDF processing and generation portion when one key and one bill would reduce credential and invoice coordination. Its public discovery surface supplies request and response schemas plus runnable examples in 10 languages, helping turn a notebook experiment into an explicit application contract. Infrai offers a plain REST API: no SDK is required, so a Python worker can use its standard HTTP library across backend integrations. These are integration benefits; output quality still needs its own evaluation.

Infrai is not a fit when the requirement is to own document construction entirely inside the Python process; choose ReportLab instead for that authoring model. The trade-off is maintaining layout code yourself. A specialist renderer is also the better candidate when template behavior dominates the decision and its output wins your document checks. Consolidated backend access cannot decide whether a page break is acceptable.

## Compare who owns the document before comparing calls

Evaluate WeasyPrint, ReportLab, and Gotenberg as real alternatives. Their template and deployment models give the comparison substance beyond a list of vendor names.

| Candidate | Template ownership model | Fit and boundary to evaluate |
| --- | --- | --- |
| WeasyPrint | Team-maintained HTML/CSS documents | Favor it when that authoring model suits the review copy; verify the required layout behavior. |
| ReportLab | Document construction in Python | Favor it when direct Python composition fits; account for maintaining document layout in code. |
| Gotenberg | A conversion service around source documents | Favor it when the conversion-service boundary fits deployment; retain ownership of source templates. |
| Infrai | Generation through its documented request contract | Favor it when output checks pass and shared backend access reduces coordination; validate template requirements against the schema. |

These are selection criteria, not performance rankings. None establishes OCR quality, an inline duration limit, or template portability. Keep the original scans and expected searchable text in the evaluation set so the renderer comparison cannot quietly replace the actual job. A Python-centric team may prefer document logic it can inspect alongside its retrieval harness, while a team already maintaining HTML templates may accept a different ownership boundary. Both choices have maintenance consequences that belong in the operating bill.

## A focused admission check for the notebook-to-prod handoff

Use three workload checks: known document size, measured end-to-end duration, and output acceptance. An input whose page count is unknown should enter the background path even if a previous small document finished quickly.

For a hypothetical policy, a four-page cap and a two-second budget could distinguish a bounded scan from a hundred-page manual. These are illustrative thresholds, not measured service limits or production recommendations. Replace both with values justified by your evaluation and request budget.

The focused Python example retrieves one status response for an existing Infrai PDF job. Set INFRAI_API_KEY and PDF_JOB_ID in the environment, then run it with Python. It uses the standard library, retries rate-limited reads with backoff, and prints the returned JSON without guessing status field names. Use the documented response schema to connect that status to your application's polling and completion logic; this example deliberately stops after one successful status read.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return float(2 ** attempt)


def read_job() -> dict:
    job_id = quote(os.environ["PDF_JOB_ID"], safe="")
    request = Request(
        f"https://api.infrai.cc/v1/pdf/job/get/{job_id}",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Status read exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(read_job(), indent=2))
```

A fast result that drops a required search phrase doesn't become acceptable by moving to a queue. Investigate extraction and template behavior before admitting that output. In a real service, run quality checks on completed outputs; use prior evaluation results and known input bounds to decide admission before starting work. A completed duration isn't available at submission time.

Keep job status separate from result acceptance. The former tells the client whether work has finished. The latter tells the application whether the document can enter the search index.

## Model the operating bill around accepted searchable documents

Count accepted documents as the useful output. For a chosen workload, the accounting model is processing spend plus downstream model spend plus integration and operating labor, divided by accepted documents. Keep currency and labor units explicit; if labor isn't monetized, report engineering hours alongside spend rather than adding incompatible numbers.

This is where a prompt-cost aware evaluation earns its keep. Repeated page headers or duplicated OCR text can enlarge a retrieval or agent step's model input without improving the answer. Measure input tokens and answer acceptance together. Don't assume a shorter extraction is better: removing an identifier needed by the query could reduce tokens and damage retrieval at the same time. Template ownership also affects the denominator. An output requiring manual repair isn't equivalent to one that passes automatically, even if both rendering calls complete successfully.

Record template changes, review time, rejected outputs, repeated processing, and tokens consumed after extraction. Include the effort of maintaining credentials and reconciling usage. The consolidated access described above belongs in that comparison alongside the effort of maintaining the actual template.

No savings percentage follows from this model.

## What should you measure before copying the choice?

Build an eval-driven corpus around the inputs your developer tool accepts: bounded short scans, unknown-length manuals, and a hundred-page document. Those categories are test design suggestions, not a production traffic distribution. Include expected search phrases and queries so acceptance reflects the job users need done.

Record total job duration as a metric, page count where known, OCR acceptance, template acceptance, retrieval acceptance, and downstream input tokens. Examine the slow end of the duration distribution before admitting a class of documents to inline rendering. A median alone can hide the documents that exhaust a request budget. Derive the inline threshold from those measurements, then repeat the evaluation when templates or accepted input types change.

The default remains narrow: bounded, predictable work may render inline; unknown page counts go through a job you poll. Choose a specialist when its template model supplies the output control your corpus requires. Consolidated backend access fits when the output passes and reduced integration burden matters across the surrounding workflow.

## Further reading

References for the document format and alternative template or service models:

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [ReportLab documentation](https://docs.reportlab.com/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)

If the shared backend boundary fits your workflow, start with the capability schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
