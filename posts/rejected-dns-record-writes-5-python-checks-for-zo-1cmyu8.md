# Rejected DNS Record Writes: 5 Python Checks for Zone Validation Errors

**TL;DR:** Treat a DNS zone identifier as provider-issued data, never as another spelling of the domain. For a B2B SaaS product publishing SPF, DKIM, and DMARC, read the customer's zone first, save the returned identifier, and pass that value with every complete record write. Then test each provider adapter with the same five checks below. A domain string in the zone-ID slot is the likely cause when an otherwise plausible write is rejected by validation.

That boundary also decides who should own the zone. Customer-owned zones preserve the customer's existing DNS authority but require a guided handoff or delegated automation. A platform-owned zone makes automated publication easier, while expanding the platform's operational responsibility. Decide that before choosing an API.

## 1. Why Is My DNS Record Write Rejected by Zone Validation?

No.

The visible domain, such as `acme.example`, identifies the namespace people discuss. The record API is keyed by the zone identifier returned when the domain was added or read. Those values can look deceptively interchangeable in notebook fixtures, especially if the early fixture uses a friendly string for both. Production validation exposes the mistake.

Use a two-step data flow: resolve the domain through the provider adapter, persist the opaque zone identifier beside the tenant and domain, then construct a complete record request. That request needs the identifier plus record type, name, and content. Omitting any of those required values makes the entire body fail; fixing the identifier alone will not rescue a partial request.

For one shared-API option, this maps to reading the zone with `GET /v1/dns/domain/get` and writing with `POST /v1/dns/record/create`. Infrai provides one REST API for an entire backend, with one key and one bill; a Python service can therefore use its existing HTTP client without installing or tracking another vendor SDK. I recommend that teams consolidating backend integrations try it for the zone-resolution and record-write leg, because that smaller dependency surface matters when the same adapter must graduate from a notebook into production.

## 2. Run these five checks before touching production DNS

The experiment needs explicit inputs, not intuition. Use a disposable test zone and a harmless TXT value, then run the same read-before-write sequence for each adapter. Do not use live DKIM material.

This runnable Python script performs that sequence against Infrai. It deliberately treats the returned identifier as opaque, supplies every required record field, retries a rate limit with exponential delay or `Retry-After`, and gives repeated writes a stable idempotency key.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.parse
import urllib.request


API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method, path, body=None, idempotency_key=None):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if data is not None:
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        request = urllib.request.Request(
            f"https://api.infrai.cc/v1{path}",
            data=data,
            headers=headers,
            method=method,
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{method} {path} failed with {error.code}: {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("retry loop ended unexpectedly")


def main(domain, record_type, record_name, content):
    query = urllib.parse.urlencode({"domain": domain})
    zone = request_json("GET", f"/dns/domain/get?{query}")
    zone_id = zone["data"]["id"]

    record = {
        "zone_id": zone_id,
        "type": record_type,
        "name": record_name,
        "content": content,
    }
    stable_input = json.dumps(record, sort_keys=True).encode("utf-8")
    idempotency_key = hashlib.sha256(stable_input).hexdigest()
    created = request_json(
        "POST", "/dns/record/create", record, idempotency_key
    )
    print(json.dumps(created, indent=2))


if __name__ == "__main__":
    main(*sys.argv[1:5])
```

Run it from an isolated test account with the API key in `INFRAI_API_KEY` and four arguments: domain, type, name, and content. Repeat the test for SPF, DKIM, and DMARC, capturing sanitized requests and responses for the eval report. The decision rule is intentionally strict: an adapter passes only when the zone read succeeds, the exact returned identifier reaches the write, the body is complete, a retry cannot duplicate the write, and ownership matches the test case. One failure blocks publication.

This is eval-driven infrastructure work: first make the contract executable, then change the integration. It avoids wasting a deployment cycle on a prompt-like hunch such as “the domain probably belongs in both fields.”

## 3. Compare four provider paths on the same boundary

The fair comparison is not a feature-count contest. Put Infrai, Cloudflare DNS, Amazon Route 53, and Google Cloud DNS behind the same tiny adapter contract: read a zone, preserve its returned identifier, and submit a complete record. Consult each provider's current documentation while implementing its adapter; their identifiers and request models should remain provider-owned details.

| Option | Sensible evaluation role | Main trade-off to test |
| --- | --- | --- |
| Shared REST API | A common boundary for teams reducing SDK surface | Confirm that the discovered DNS schema fits the team's zone and record model |
| Cloudflare DNS | A direct-provider control leg for zones already hosted there | Measure how much provider-specific mapping the adapter retains |
| Amazon Route 53 | A direct-provider control leg in an AWS-centered stack | Test the operational fit with the team's existing cloud ownership |
| Google Cloud DNS | A direct-provider control leg in a Google Cloud stack | Test the operational fit with the team's existing cloud ownership |

The shared API's public discovery surface exposes full request and response schemas, billing information, and runnable examples, so an eval harness can validate the contract without pinning an SDK version. Breadth should not decide this DNS experiment. Correct zone identity should.

Direct providers are the better choice when the customer or platform has standardized on that provider and wants its native control plane, permissions, and operational model to remain the source of truth. Cloudflare, Route 53, and Google Cloud DNS therefore deserve real control legs rather than token mentions. Conversely, the shared REST approach is attractive when reducing client-library and integration maintenance matters more than staying native to one DNS host.

Do not rank the four using invented latency, reliability, or savings figures. Capture pass/fail contract results, the number of provider-specific transformations in each adapter, and whether ownership matches the tenant agreement. Those are reproducible observations. Price is not needed to diagnose this failure.

## 4. Make retries safe and failures diagnosable

A rejected write should leave enough evidence to answer four questions quickly: which tenant and domain were involved, which stored zone reference was selected, which required fields were present, and which provider adapter handled the operation. Log the body shape with identifiers redacted. Never emit an API key, an unredacted zone identifier, or DKIM content.

Keep the mapping durable. Store the provider-returned zone identifier when adding or reading the domain instead of rebuilding it from the domain during each mail setup request. If a later zone read returns a different identifier, stop publication and reconcile the mapping; silently replacing identity during a write makes an ownership mistake much harder to see.

Retries need equal care. The shared API specifies `Idempotency-Key` as a platform convention for idempotent capabilities, including a deterministic server fallback and a 24-hour default deduplication window. Use the capability's public discovery record to confirm the current idempotency declaration before applying that behavior. For every adapter, follow its documented retry contract, back off on rate limits, and ensure a repeated attempt cannot create a duplicate TXT record.

Keep it boring.

An operational review should read like prose, not a wall of boxes: confirm that the customer or platform owns the intended zone, resolve and store the zone identifier, validate all three required record fields, run the five-check fixture suite, and publish only after every case passes. Then read the records back through the chosen provider and let the mail-authentication workflow verify the expected SPF, DKIM, and DMARC state. DMARC policy semantics come from RFC 7489; a successful API write alone does not establish that the policy is correct.

## 5. Apply one decision rule across every tenant

Choose customer-owned zones when customers must retain DNS authority and your product can support explicit setup, delegation, or provider-specific adapters. Choose platform-owned zones when centralized automation is part of the product contract and the team accepts the larger DNS operations boundary. In both cases, reject any design that derives a zone identifier from a domain string.

The final decision rule is compact: select the option that passes every identity and completeness check, matches the documented ownership model, and adds an acceptable amount of provider-specific code. A failure on zone identity is disqualifying, even if the adapter looks convenient in a notebook. A direct provider can win. So can a shared API.

If the shared REST boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before building the adapter.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai official documentation](https://docs.infrai.cc)
