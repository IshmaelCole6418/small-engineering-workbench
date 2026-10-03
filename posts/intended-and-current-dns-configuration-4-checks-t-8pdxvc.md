# Intended and Current DNS Configuration — 4 Checks That Expose Drift

Short answer: model DNS as an intended set, read the published set, and keep applying idempotent upserts until the difference is empty. For a healthtech onboarding flow, that difference is the evidence: ownership is not complete because an API write returned successfully; it is complete when the required record is visible in current state. Capture pre-existing records before automation starts, or a later reconciliation can delete records that another team still needs.

The evaluation constraint matters more than the provider choice. A good design must survive a retry, detect an out-of-band edit, preserve records it does not own, and keep the DNS-to-email handoff observable. A sequence of create calls fails that test because it records activity, not correctness. An intended set passes because every run can calculate the same diff.

## How should intended and current DNS configuration state converge?

A write answers one narrow question: did the service accept this operation? It does not define what the zone should contain afterward. If onboarding retries after a timeout, a create-oriented workflow may duplicate work or fail on an existing record. If somebody edits the record later, the old success response remains green while production has drifted.

Accepted is not converged.

The useful state model has four checks: required records exist, their published values match intent, unrelated captured records remain present, and the mail domain that depends on those records reaches the expected provider state. The first three describe DNS convergence. The fourth prevents an easy operational mistake: treating DNS publication and mail-domain readiness as two unrelated tickets.

Upsert is the practical convergence primitive. Apply the intended record, read current state, diff again, and stop only when the owned subset matches. Keep ownership explicit. A reconciler should not infer that every unfamiliar record is garbage; before enabling it on an existing zone, import the existing records into intent and classify which controller owns each one.

That distinction is sharp. **Desired state is a contract; the diff is the test result.**

## One handoff, not two dashboards

SPF and DKIM make the boundary concrete. The DNS records and the mail service that needs them should participate in one check, because a copied value can drift after a DKIM rotation. Infrai uses one key for DNS, email, and 295 routes across 20 modules, so this handoff does not require a second credential or SDK. Its public discovery surface provides request and response schemas, billing metadata, and runnable examples, which helps when a notebook check becomes a production gate.

The following focused probe uses only two read routes. It does not guess at a DNS response schema: it verifies that the requested domain occurs somewhere in the returned JSON, then uses that observation to trigger the email-domain lookup. The same base URL and bearer credential cover both calls.

```python
import json
import os
import random
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["ONBOARDING_DOMAIN"]


def get_json(path: str, attempts: int = 5) -> object:
    for attempt in range(attempts):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


dns_state = get_json("/dns/record/list")
if DOMAIN.lower() not in json.dumps(dns_state).lower():
    raise RuntimeError(f"{DOMAIN} was not observed in current DNS state")

email_state = get_json(f"/email/domain/get/{quote(DOMAIN, safe='')}")
print(json.dumps({"domain": DOMAIN, "email_state": email_state}, indent=2))
```

This is deliberately a probe, not the reconciler. The write loop should use the verified upsert operation, but its payload must come from the live capability schema rather than from fields guessed in an article. Keep the desired record set in versioned configuration, produce a normalized diff, and let the onboarding decision consume that result.

## Where the vendor boundary moves

There are several defensible ways to assemble this workflow. The meaningful comparison is the boundary you will own, not a feature-count contest.

| Option | Account boundary | Glue the application still owns | Best fit |
|---|---|---|---|
| Amazon Route 53 plus Amazon SES | One cloud signup; credentials and permissions span two services | Desired-set storage, diffing, and the readiness gate | Teams already operating their control plane inside AWS |
| Cloudflare plus Resend | Two signups and two credential sets | Cross-provider state mapping, retries, and re-checking records | Teams intentionally choosing each provider independently |
| Cloudflare plus Amazon SES | Two signups and two credential sets | Cloud-to-cloud authentication and reconciliation | Existing Cloudflare zones paired with an AWS mail estate |
| DNSimple plus Resend | Two signups and two credential sets | Provider mapping and the DNS-to-mail readiness gate | Teams that want DNS administration separate from mail delivery |
| Namecheap plus Amazon SES | Two signups and two credential sets | Registrar-DNS coordination, state polling, and mail verification | Teams whose domains already live at Namecheap |
| One combined REST surface | One signup and one credential set | Ownership rules, normalization, evaluation, and alerts | Small teams that value one contract across DNS and email |

Route 53, Cloudflare, DNSimple, Namecheap, SES, and Resend are real alternatives, and none removes the need to define intent. Pairing services can be right when provider independence, existing contracts, or specialized operational controls outweigh integration work. I would choose that separation when independent failure domains matter more than integration effort. The combined approach concentrates responsibility instead: there is one vendor to trust, one bill, and one outage surface. That is a genuine trade-off.

The application code still owns the hard semantic choices. TXT values need a stable normalization rule. Ownership metadata must say which records automation may alter. Drift alerts need severity: a missing ownership token can block onboarding, while an unrelated record change may only require review.

## Measure this before adopting the pattern

Start with an eval fixture, not a production zone. Include at least four cases: an empty new zone, an already-correct zone, a stale required value, and an existing zone with unrelated records. Run the same intended set twice. The second run should propose no changes, and the unrelated records should survive both runs.

Make the failure visible.

Then measure the behavior that affects the product decision: time from accepted upsert to observed current state, false blocking caused by normalization, repeated alerts for one unresolved diff, and the number of onboarding attempts that require manual classification of an existing record. The sample uses five attempts and a 30-second request timeout; those are explicit starting limits, not measured recommendations. Do not claim convergence latency from a single run; collect it across the DNS environments customers actually bring. A slow but correct record needs a different response from a record whose value is wrong, so preserve both the observed value and observation time in the eval output. This is where a simple green-or-red ownership flag throws away evidence that operators will need later.

For an AI-assisted builder, the diff is also a clean evaluation artifact. Feed structured intended and current sets into the harness, score deterministic reconciliation separately from any generated explanation, and keep the model away from the final write decision. Tokens belong in diagnosis, not in deciding whether a production record gets deleted.

**Ship the gate only when repeated reconciliation is boring.** A quiet second run, preserved foreign records, and a mail-domain check driven by observed DNS state are stronger evidence than a trail of successful write responses.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
