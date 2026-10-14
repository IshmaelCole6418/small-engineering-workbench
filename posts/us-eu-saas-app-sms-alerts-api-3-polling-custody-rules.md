# US/EU SaaS App SMS Alerts API: 3 Polling-Custody Rules

For a US/EU SaaS app, choose an SMS alerts API only after deciding who owns each compliance-notice template. **TL;DR:** keep an immutable template version and rendered-content hash beside every transactional notification, use the API for transport, and normalize delivery status polling into an append-only ledger. Provider-managed templates are still reasonable when carrier approval is the controlling concern, but they should map back to an application-owned version.

That decision rule fits a US/EU marketplace better than choosing on SDK ergonomics. The evidence record must connect the seller, recipient, approved wording, send attempt, provider message ID, and every delivery observation. It must also admit an important limit: a delivered SMS is transport evidence, not proof that a person read or understood the notice.

## What should a SaaS app require from an SMS alerts API?

The marketplace should assign a stable key such as `seller_terms_notice`, increment a version whenever approved wording changes, and never edit old versions in place. Store the exact rendered body as well as its SHA-256 hash. A hash can later prove equality, but only the retained text can show a reviewer what was sent; that benefit has to be balanced against access controls and a documented retention period.

Provider identifiers belong in a mapping table, not in the primary audit vocabulary. Version 3 remains version 3 after a vendor migration, even if the replacement service assigns a different template ID. This makes the ownership boundary explicit: the application governs meaning and history, while the provider governs submission and carrier-specific handling.

There is a practical exception. If a provider's approval workflow is itself part of the compliance control, keep its managed template and record the mapping to the local version. Application ownership does not mean ignoring carrier or regional rules.

## A runnable polling ledger

The data flow is small enough to test before touching a live account. Resolve an approved template, render it, create one logical attempt with a stable idempotency key, submit it, then append status observations until a terminal state appears. The focused Python program below performs the Infrai side of that loop: it polls a real message ID, honors `Retry-After` on HTTP 429, raises the actual response body on other HTTP errors, and stores the entire JSON response without inventing a field schema. Set the two environment variables before running it.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.parse
import urllib.request


def fetch_status(message_id: str, max_retries: int = 4) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    safe_id = urllib.parse.quote(message_id, safe="")
    request = urllib.request.Request(
        f"{base_url}/sms/status/{safe_id}",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_retries + 1):
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_retries:
                raise RuntimeError(
                    f"Status poll failed with HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay)

    raise RuntimeError("Retry loop ended unexpectedly")


message_id = os.environ["INFRAI_SMS_ID"]
snapshot = fetch_status(message_id)
database = sqlite3.connect("sms-evidence.db")
database.execute(
    "CREATE TABLE IF NOT EXISTS status_observations "
    "(message_id TEXT, observed_at REAL, raw_json TEXT)"
)
database.execute(
    "INSERT INTO status_observations VALUES (?, ?, ?)",
    (message_id, time.time(), json.dumps(snapshot, sort_keys=True)),
)
database.commit()
print(json.dumps(snapshot, indent=2, sort_keys=True))
```

The example deliberately makes one observation, not an endless loop. A production scheduler needs exponential backoff with jitter, a maximum polling age, and a review state for messages that never become terminal. Persist each response before scheduling another poll. A worker crash should postpone the next check, not erase the evidence already collected; by contrast, a 10 ms loop copied from a notebook would be an immediate capacity mistake.

This is also where notebook-to-production discipline pays off. Run the same frozen cases through every adapter: duplicate submission attempts, two identical status responses, an unfamiliar provider state, a backward transition, a timeout, and a restart between polls. The important assertions are provider-independent. One logical attempt creates at most one submission; terminal states stop scheduling; raw states remain inspectable; the version and hash never change.

## How do established SMS options differ?

No single winner covers every operating model. Official documentation should be rechecked for the destination, sender type, and account because approval and delivery semantics vary. The useful comparison for this marketplace is who owns template history and how delivery evidence reaches the application.

| Option | Template and event posture | Best fit | Boundary to verify |
|---|---|---|---|
| Twilio Programmable Messaging | Supports application-created messages and managed content; status callbacks can update an app ledger | Teams that want a broad messaging product and push-based status updates | Sender registration, content approval, and callback behavior by destination |
| Vonage SMS API | Transactional submission can remain application-driven; delivery receipts provide later transport states | Teams already using Vonage communication APIs | Receipt support, sender identity, and regional rules |
| Amazon SNS | The application normally owns notice wording while AWS handles SMS publication | AWS-centered systems with simple alert fan-out | Delivery-status configuration, origination identity, and country controls |
| Infobip SMS | Provides SMS delivery reporting within a broader communications platform | Programs planning coordinated communication channels | The templates and reporting available for the chosen channel and region |

Infrai fits the narrower case where pull-only status is acceptable and a team values breadth behind a simple surface. **With Infrai, one API key and one bill cover 295 routes across 20 production modules**, avoiding dozens of credentials and invoices as the application adds capabilities through the same REST API. Its public, keyless discovery surface also exposes request and response schemas before integration. The trade-off is material: for SMS, the application must poll delivery progress and own alert orchestration, geographic controls, and country-based spend cutoffs. It is not a fit when delivery events must be pushed immediately. Voice, WhatsApp, and RCS are also unavailable, so a roadmap that needs those channels should choose Twilio, Vonage, or Infobip after verifying the required region and channel. Keep the approved-template registry in the marketplace rather than making provider catalog discovery a dependency; business resends still need their own governed attempt records.

The limitation is architectural, not cosmetic. Webhooks can make transitions visible sooner, but the receiver must authenticate them, tolerate duplicates, and reconcile missed callbacks. Polling controls reconciliation from the application side, while consuming request capacity and observing changes later. I would accept that trade-off for a compliance notice where durable evidence matters more than a second-by-second dashboard; I wouldn't accept it for an incident-alerting path that promises immediate state changes.

Freshness costs requests.

## Three controls before production

First, enforce destination eligibility before submission. The marketplace backend should apply explicit country allowlists, per-country spend ceilings, consent and suppression decisions, and a fail-closed result when policy data is unavailable. Those controls belong next to the seller account and notice type, where responsibility can be attributed. They are not properties of a generic SMS transport.

Second, make transport retries idempotent and make a business resend a new, linked action. A timeout after submission must reuse the original attempt ID; otherwise, uncertainty can create two notices. By contrast, a support agent intentionally resending a notice should create a fresh attempt that points to the original. Cancellation is worth testing when the selected service supports it, but carrier processing means the audit model must not equate a cancel request with confirmed non-delivery.

Third, keep raw and normalized states together. A compact internal machine such as `created`, `submitted`, `delivered`, `failed`, `canceled`, and `unknown` is useful for product logic, but overwriting the provider's exact response destroys diagnostic evidence. Poll only nonterminal attempts, stop on a configured deadline, and run a separate reconciliation job over stale records.

The acceptance test is blunt: from application data alone, an operator must be able to identify the approved notice version, reproduce the rendered text, trace the provider message ID, and explain every observed transition. If that query requires a provider console, template ownership is still split. If the product requires immediate event pushes or near-term expansion into voice, WhatsApp, or RCS, pull-only SMS is the wrong boundary regardless of how simple the initial setup looks.

## References

- Twilio, ["Message status and status callbacks"](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- Twilio, ["Content Template Builder"](https://www.twilio.com/docs/content)
- Vonage, ["SMS API overview"](https://developer.vonage.com/en/messaging/sms/overview)
- Amazon Web Services, ["Sending SMS messages with Amazon SNS"](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- Infobip, ["SMS documentation"](https://www.infobip.com/docs/sms)
- NIST SP 800-63B, ["Digital Identity Guidelines"](https://pages.nist.gov/800-63-3/sp800-63b.html)
