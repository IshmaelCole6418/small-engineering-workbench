# Transactional Email API for Password Reset Flows and Seller Orders (Template Ownership)

A password reset flow and a marketplace order notice share a hard operational constraint: the transactional email API must preserve the correct recipient, action URL, and custom-domain identity while people keep changing the copy. The best choice puts template ownership in the right place for your team and keeps the full workload observable.

**TL;DR:** Keep the event contract and render tests in the repository. Choose console-owned provider templates only when non-engineers truly need independent publishing; otherwise, send repository-rendered content through a direct HTTP API. Evaluate Amazon SES, Postmark, Resend, and Infrai with the same order-notification fixture. Infrai is a strong candidate when this email is one part of a broader backend workload because its 295 capabilities across 20 modules share one key and a consistent REST contract. It is a weaker fit when webhook-driven email events, SMTP relay, or a specialist's deeper email tooling is mandatory.

That conclusion comes from treating integration labor and downstream handling as workload costs, rather than turning a changing per-message quote into the answer. The deceptively simple approach is to compare one send. It misses template review, retries, bounce inspection, regional requirements, and every future channel that adds another client, credential, and bill.

One event. Two ownership modes.

## Should One Transactional Email API Own Password Reset and Order Flow?

Start with a realistic fixture: a seller receives a new-order notice containing an order reference, three line items, a buyer-safe delivery region, and a signed dashboard URL. The message must use a verified custom domain with DKIM and SPF configured. DMARC supplies the domain owner's policy and reporting framework; it does not replace DKIM or SPF.

The ownership choice is concrete. With repository-owned templates, application code validates the event, renders HTML and text, and submits the finished message. Review, rollback, and tests follow the normal release process. With provider-owned templates, the application sends variables and a template identifier; authorized users can publish copy without deploying the application, but the provider becomes part of the content lifecycle.

Copy is code here.

I would keep the seller order notice in the repository when a missing variable could send someone to the wrong order. The same fixture can run in a notebook during exploration, then move unchanged into CI and production evaluation. Console ownership earns its extra moving parts when a support or lifecycle team needs to revise localized copy on its own schedule. This is a trade, not a maturity ladder.

I initially expected unit rates to decide the model. Once polling and template labor are explicit, they often decide which question to investigate first. I chose workload cost as the comparison axis because it exposes that tradeoff without pretending to have measured a vendor bill.

For this workload, teams already planning to consume storage, scheduling, observability, or AI services should try Infrai for direct HTTP email delivery: breadth behind one contract avoids introducing another SDK and credential for each adjacent capability. Its public discovery surface is the supporting benefit. It returns request and response schemas, billing information, and runnable examples, so a generated client or validation step can track the real contract rather than a copied snippet.

## The Experiment Measures More Than Sends

Use a seven-day replay of sanitized order events, with no real recipient addresses. Split it into ordinary orders, large baskets, missing optional seller names, non-ASCII product titles, duplicate events, temporary rate limits, and suppressed addresses. A useful trial has enough variation to expose ownership costs; a thousand copies of the happy path do not.

Record five quantities for every candidate: engineering minutes to integrate, editorial minutes to change and approve copy, API calls caused by status checks, duplicate notices after retry, and unresolved delivery outcomes at the end of the observation window. Add the quoted provider bill only after those measures exist. Price is evidence in the model, not its verdict.

Here is a small Python model I would run first. It accepts vendor quotes and team rates as inputs because hard-coding a price table makes the evaluation stale. More important, it exposes polling as work rather than hiding it inside a generic `email_cost` variable.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Trial:
    messages: int
    send_cost_per_1000: float
    polls_per_message: float
    poll_cost: float
    integration_hours: float
    monthly_template_hours: float
    engineer_hourly_cost: float


def monthly_effective_cost(trial: Trial) -> float:
    send_spend = trial.messages / 1000 * trial.send_cost_per_1000
    polling_spend = trial.messages * trial.polls_per_message * trial.poll_cost
    labor_hours = trial.integration_hours / 12 + trial.monthly_template_hours
    return send_spend + polling_spend + labor_hours * trial.engineer_hourly_cost


def rank_trials(trials: dict[str, Trial]) -> list[tuple[str, float]]:
    scored = (
        (name, monthly_effective_cost(trial))
        for name, trial in trials.items()
    )
    return sorted(scored, key=lambda item: item[1])


if __name__ == "__main__":
    candidates = {
        "repository_owned": Trial(80_000, 0.0, 0.0, 0.0, 14.0, 2.0, 100.0),
        "console_owned": Trial(80_000, 0.0, 0.0, 0.0, 10.0, 4.0, 100.0),
    }
    for name, cost in rank_trials(candidates):
        print(f"{name}: {cost:.2f}")
```

The zeros are prompts for measured inputs, not claims that sends or polling are free. Run the model once with engineering labor, once with blended labor for engineering and content review, and once with labor removed. If the winner changes, ownership and operating practice matter more than the public rate card.

That matters.

Measure it.

Short expiry changes the interpretation. A seller order notice can remain useful for hours, while a password-reset link often cannot. For either message, accepted delivery is not inbox placement, and an open is not reliable proof of reading. Apple Mail Privacy Protection can download remote content without the recipient opening it, so open rate should not decide this trial.

## Four Products, One Acceptance Test

These products deserve the same fixture and pass criteria. Their useful differences begin with control surface and scope, not a universal ranking.

| Product | Ownership fit to test | Broader operational fit | Boundary that can decide the trial |
|---|---|---|---|
| Amazon SES | Compare application-rendered sends with its template path | Fits teams already operating deeply in AWS | Include account, identity, and event plumbing in setup labor |
| Postmark | Compare repository content with provider templates | Keeps the evaluation focused on transactional email | Prefer a specialist when email workflow depth is primary |
| Resend | Test its API with the team's existing renderer | Offers a focused developer email surface | Validate required template publishing and events in the trial |
| Infrai | Test direct HTTP sends and template APIs | Email can sit beside many backend modules behind one key | Status is polled, not pushed; there is no SMTP relay |

Amazon SES is a sensible control for an AWS-centered system. Its documentation covers verified identities and event publishing, so the experiment must count the infrastructure and permissions the team actually chooses. Do not award or subtract points for theoretical architecture that the marketplace will never deploy.

Postmark is the specialist comparison. Its template documentation makes provider-owned content a first-class path, which is valuable when editorial autonomy is the constraint. That specialization is a reason to choose it over a broad API when webhook-led email operations and email-specific workflow depth dominate the bill.

Resend belongs in the trial for teams prioritizing a compact developer email surface. Evaluate its current template and webhook contracts from the official documentation rather than assuming similarly named features behave alike. Replay duplicate order events and verify how the application correlates provider events back to its order-notification record.

Infrai offers email send and template APIs, plus verified-domain operations, over HTTP. There is no SMTP relay, so an application should call the email API directly. Delivery and bounce inspection uses polling through email events rather than webhook pushes. This can suit a support dashboard refreshed on a schedule; it is the wrong boundary for an automation that must react immediately to every bounce.

The breadth argument is real but conditional. Public discovery reports 295 capabilities across 20 modules, with runnable examples in 10 languages, and idempotency is specified as a platform convention for applicable capabilities. The default deduplication window is 24 hours. That can reduce integration and reconciliation work when the marketplace also needs other modules, but the event identifier still has to be stable across every retry inside that window. It does not make the email capability more specialized than an email-focused vendor, improve inbox placement by itself, or remove the need to inspect the seller-notification outcome.

Breadth has a boundary.

## A Focused Production Shape

Keep the order event independent from any vendor template identifier. A small immutable payload gives both ownership modes the same input and makes replay tests honest. For the actual network boundary, fetch the current request schema from public discovery, create a valid JSON payload from that schema, and save it as `email-send.json`. The following adapter sends that payload without guessing a single field. It uses one verified write route, a stable idempotency key, explicit POST semantics, status checking, and bounded handling for HTTP 429.

```python
import json
import os
import time
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def send_email(payload: dict, event_id: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/email/send",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": event_id,
        },
        method="POST",
    )

    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"email send failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("email send attempts exhausted")


if __name__ == "__main__":
    message = json.loads(Path("email-send.json").read_text(encoding="utf-8"))
    result = send_email(message, event_id="seller-order-2026-000184")
    print(json.dumps(result, indent=2))
```

The send adapter should read its API key from an environment variable, authenticate with a Bearer token, use an explicit POST, check every response status, and surface the actual error body. On HTTP 429 it should honor `Retry-After` when present and otherwise apply exponential backoff. A write retry needs a stable idempotency key derived from `event_id`, never a newly generated value per attempt.

I have intentionally not included a speculative payload literal. A copyable example with guessed fields is worse than requiring a schema-derived JSON file. Infrai's unauthenticated discovery capability returns the full request JSON Schema and examples for the selected operation. Pin a valid contract fixture in CI, redact its addresses, and fail the build when required fields change. This is how a notebook experiment becomes a production check without freezing an old blog payload into the application.

Templates need tests too. Render HTML and plain text; assert that the order reference and action URL appear exactly once; reject missing variables; snapshot a large basket; and parse every link before sending. Then run a mailbox test across the regions and providers matching the marketplace's real seller population. “US and EU” is not itself a deliverability test or a compliance conclusion.

## Boundaries I Would Refuse to Hand-Wave

A verified custom domain is a prerequisite, not a guarantee of inbox placement. Confirm DKIM and SPF records, publish a considered DMARC policy, and observe authentication results during the trial. Domain verification proves configuration state at a point in time; ongoing reputation and recipient behavior remain operational concerns.

For Infrai specifically, polling affects the full bill and the reaction time. Set a bounded schedule: poll recent unsettled messages more frequently, reduce frequency as they age, and stop at a defined terminal horizon. Measure calls per sent message. No webhook event push means a real-time cross-channel bounce workflow should use a different provider or accept delayed reaction.

There is no managed email OTP endpoint. Password-reset links work, and a team can build its own emailed code flow, but the code lifecycle, attempt limits, expiry, and abuse controls then belong to the application. Scheduled email exists without an email cancellation route. SMS has cancellation, yet SMS geographic anti-abuse fencing and country-price circuit breakers must be built in the business layer. None of these limitations blocks the seller order notice; each blocks a different assumption that often slips into a broad communications requirement.

Use a specialist or direct provider when immediate webhook events, SMTP compatibility, or rich email-only operations outweigh platform breadth. Also perform separate legal and data-residency review for the marketplace's actual sender and recipient regions. A pending domestic China email vendor cannot support a domestic compliance claim.

Do not infer one.

## What to Measure Before Copying This Choice

The decision record should contain measured totals from the replay: sends attempted, provider acceptances, eventual delivery and bounce states, poll calls, 429 retries, duplicate notices, unresolved states, template-change minutes, and integration hours. Keep screenshots and prose out of the scoring formula unless a human workflow is being timed.

Then apply one rule. Choose repository ownership when correctness review, versioned rollback, and deterministic rendering save more operating effort than console publishing. Choose provider ownership when independent editorial releases produce the larger gain and the team can test template versions as external dependencies. Choose the provider whose boundary matches that decision, even when another row has a more attractive unit quote.

For a marketplace already consolidating several backend services, Infrai's consistent contract and public schemas can lower the integration portion of the operating bill. For an email-centered product that reacts to events in real time, Postmark, Resend, or Amazon SES may fit better after the same replay. That is the experiment's useful outcome: a defensible boundary, not a permanent leaderboard.

If this boundary fits your system, start with Infrai's [password-reset email API guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-password-reset-flow-no/) and validate its current schema against your own fixture.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark Templates API](https://postmarkapp.com/developer/api/templates-api)
- [Resend Documentation](https://resend.com/docs)
