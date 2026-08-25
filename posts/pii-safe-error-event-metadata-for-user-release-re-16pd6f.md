# PII-Safe Error Event Metadata for User, Release, Request ID, EU, and US Analysis

Attach the smallest metadata set that can explain an error across a release, environment, request, and pseudonymous tenant cohort; keep raw user identity and arbitrary application context out of the event. For a developer-tools experiment split across EU and US tenants, that gives enough structure to compare signal quality without turning error tracking into a shadow customer database.

The deciding constraint is diagnostic value, not maximum context. A simple approach sends every local variable and user property because one of them might help later. It also creates noisy dimensions, unstable dashboards, larger payloads, and a privacy review nobody can answer from memory. The better approach starts with the decisions an engineer must make: Is this new in a release? Is it isolated to an environment or cohort? Can the failing request be found in controlled logs? Is the same logical failure being counted once or hundreds of times?

**The practical rule is an allowlist plus purpose, retention, and access rules for every field.** If a field has no named debugging or evaluation use, don't attach it.

## Govern metadata as a small error-event data contract

Use four layers. The first describes the failure: a normalized error type, a stable fingerprint, the operation being attempted, and a scrubbed message or message template. The second describes deployment: service, release identifier, environment, and runtime. The third supports correlation: request ID and trace ID, provided those identifiers lead to a controlled system rather than encoding customer data. The fourth carries experiment context: experiment name, variant, tenant cohort, and the evaluation or prompt version that was active.

Identity deserves a narrower path. When per-user recurrence is genuinely needed, attach a rotating pseudonymous subject identifier produced outside the event pipeline. Don't send a name, email address, access token, prompt text, IP address, or raw internal user ID merely because the client library accepts custom fields. A tenant cohort such as `eu_small_team` is usually more useful for an experiment comparison than a tenant name. It aligns the event with the decision while reducing cardinality and exposure.

Here is a compact schema for the experiment note:

| Field | Diagnostic question | Handling rule |
|---|---|---|
| `error.type` | What class of failure occurred? | Use a normalized, low-cardinality value |
| `error.fingerprint` | Are two events the same logical failure? | Derive from stable code context, not message values |
| `service`, `release`, `environment` | Where and when was it introduced? | Populate from deployment metadata |
| `request_id`, `trace_id` | Which controlled records explain the request? | Generate opaque IDs; set a retention policy |
| `experiment`, `variant`, `eval_version` | Which tested behavior was active? | Use versioned configuration values |
| `tenant_cohort`, `data_region` | Does signal differ across the comparison groups? | Prefer coarse, documented categories |
| `subject_id` | Does one pseudonymous subject recur? | Omit by default; rotate and restrict access when used |

No field is automatically PII-safe just because its label looks technical. A request ID can become linkable through another system, and a free-form error message can absorb input values. Treat the event as a joinable record, document which joins are intended, and prevent the rest.

## Carry the contract from a Python notebook into production

Notebook code tends to accumulate rich Python dictionaries. Production telemetry should cross a much tighter boundary. Build a new event from approved inputs instead of deleting a few dangerous keys from an arbitrary object; deletion lists lose as soon as a new framework adds a field.

The following focused example creates an error event for a cohort experiment. It uses an HMAC to derive a pseudonymous subject value, but leaves that value out unless the caller explicitly enables the approved use case. The secret must come from a managed secret store and be rotated under the team's policy.

```python
from __future__ import annotations

import hashlib
import hmac
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class ErrorContext:
    error_type: str
    operation: str
    service: str
    release: str
    environment: str
    request_id: str
    trace_id: str
    experiment: str
    variant: str
    eval_version: str
    tenant_cohort: str
    data_region: str


def pseudonymous_subject(raw_user_id: str, secret: bytes) -> str:
    digest = hmac.new(
        secret,
        raw_user_id.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
    return digest[:32]


def build_error_event(
    context: ErrorContext,
    *,
    fingerprint: str,
    include_subject: bool = False,
    raw_user_id: str | None = None,
    pseudonym_secret: bytes | None = None,
) -> dict[str, Any]:
    event: dict[str, Any] = {
        "error.type": context.error_type,
        "error.fingerprint": fingerprint,
        "operation": context.operation,
        "service": context.service,
        "release": context.release,
        "environment": context.environment,
        "request_id": context.request_id,
        "trace_id": context.trace_id,
        "experiment": context.experiment,
        "variant": context.variant,
        "eval_version": context.eval_version,
        "tenant_cohort": context.tenant_cohort,
        "data_region": context.data_region,
    }

    if include_subject:
        if raw_user_id is None or pseudonym_secret is None:
            raise ValueError("subject pseudonymization requires an ID and secret")
        event["subject_id"] = pseudonymous_subject(raw_user_id, pseudonym_secret)

    return event
```

This boundary is intentionally boring.

Good.

Raw exceptions, request bodies, headers, prompts, model responses, and notebook state never enter `build_error_event`, so a later exporter can't accidentally forward them. Scrubbing still belongs downstream as defense in depth, especially for exception messages, but it isn't carrying the whole privacy design. The limitation is that a strict allowlist slows ad hoc debugging when an engineer discovers, during an incident, that an unapproved dimension would have answered the question. Accept that friction for the shared event stream; choose a separately governed diagnostic capture when the missing context is necessary and its collection has been approved.

There is a catch: pseudonymization is not anonymity. The event remains useful precisely because records can be related, so the subject field still needs a stated purpose, limited access, and deletion behavior. I'm not sure a single retention period can fit every EU and US tenant contract; legal obligations and customer commitments vary, and counsel plus the data owner should resolve that question. Engineering can make the decision enforceable by attaching a policy class or routing events to separately governed stores, rather than guessing jurisdiction from an IP address.

## Treat a false cohort winner as an evaluation failure

Count failures by normalized fingerprint, release, variant, and tenant cohort before looking at individual events. Raw event totals are a poor first comparison when traffic volume, retries, or one repeatedly failing subject can dominate a cohort. At minimum, pair the error count with an exposure denominator that matches the experiment: requests, task runs, or eligible sessions. Keep the numerator and denominator on the same cohort definition and time window.

Then inspect distribution, not just an overall average. The Core Web Vitals guidance uses the 75th percentile to assess user experience, which is a useful reminder that an aggregate can conceal a weak segment. Error experiments need their own predeclared statistic rather than borrowing that threshold blindly, but the method transfers: compare the same percentile or rate across cohorts, retain the sample-size context, and look for a release-specific shift. Your mileage may vary when cohorts are small or workloads differ sharply; in that case, extend the evaluation window or redesign the cohort before declaring a winner.

One subtle failure mode appears when a new variant emits cleaner fingerprints than the old one. Its dashboard may show fewer groups even if users encounter the same number of failures. Another appears when retries inflate event volume. Record retry attempt as a bounded numeric field only if the experiment needs it, and decide whether the primary metric counts attempts or terminal task failures. Otherwise the observability pipeline quietly becomes the experiment.

Keep prompt and token concerns visible without logging prompt content. A versioned prompt identifier, model configuration identifier, input-token bucket, and output-token bucket can support cost and quality analysis with less exposure than raw text. Buckets trade precision for lower cardinality and reduced reconstruction risk. They are not suitable when exact billing reconciliation is the job; use the governed billing record for that purpose and keep error events focused on diagnosis.

Fast feedback matters, too. Add schema checks to the eval harness so a notebook experiment fails before deployment if it introduces an unknown event key, a raw identifier, or a free-form experiment label. A test can also assert that two exceptions with different customer text produce the same fingerprint. Small tests catch expensive telemetry drift.

## Test deletion and storage migration before launch

A metadata design isn't finished when ingestion works. Run a deletion drill against the same joins engineers use during diagnosis: start with the governed subject mapping, locate eligible event records, remove or detach them under the applicable policy, and verify that cached exports don't silently preserve the link. This is also where opaque request IDs earn their keep. They can correlate controlled systems without embedding an email address or tenant slug in every copy.

Keep cohort-level aggregates separate from linkable event records. The trade-off is reduced drill-down after the detailed records expire, so preserve the schema version, cohort definition, denominator, and aggregation method needed to interpret the experiment result. Don't preserve raw identity merely to make an old dashboard clickable.

## What metadata should teams attach to user error events across releases?

Before copying this pattern, write down the experiment decision, expected event volume, cohort denominator, allowed dimensions, access group, retention rule, deletion path, and the owner who can change the schema. Measure missing request IDs, unknown cohort values, fingerprint cardinality, duplicate-event rate, and the share of events that join successfully to controlled traces. Those checks reveal whether metadata improves signal or merely creates the appearance of precision.

The approach is not suitable when the debugging question requires inspecting full customer content, such as reproducing a content-specific parsing failure. In that case, choose a separate, consented, access-controlled diagnostic workflow with its own retention and audit rules; don't stretch the general error stream until it holds sensitive payloads. Likewise, omit `subject_id` when release, request, and cohort metadata answer the operational question. Less data is often the sharper instrument.

Ship the schema with the experiment, review it like an API, and remove fields whose purpose expires. The winning cohort should be the one with better evaluated behavior and a trustworthy error rate, not the one whose telemetry happens to be louder.

## References

- https://web.dev/articles/vitals
