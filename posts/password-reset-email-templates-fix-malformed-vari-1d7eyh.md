# Password Reset Email Templates: Fix Malformed Variables and Placeholder API Errors

Short answer: for a fintech password-reset email, choose an API template flow when you can preview and validate every placeholder before sending; choose an SMTP relay when your team needs mail-server controls or an existing compliance pipeline. A malformed template is an application error to catch before delivery, not a reason to swap transports.

The data flow is small: create a versioned reset template, preview it with a realistic `reset_link` and `user_name`, validate the same variable set in your backend, then send only the rendered result. I keep that path close to the eval harness I use for RAG work: a known fixture, a visible output, and a test that fails before production traffic.

## How should you debug a password reset email with missing variables?

Start with the template, not the inbox. A missing placeholder such as `reset_link` can produce broken HTML even when the send request is authenticated. A malformed payload can also be rejected by the API, which is useful feedback if your service records the response body and request ID.

Here is a compact Python flow using the documented template create, preview, update, and send paths. The field names are deliberately kept together in one payload so a schema change is easy to review. The example uses a client idempotency key for the send and retries rate limits with `Retry-After` when available.

```python
import os
import time
import uuid
import requests

BASE_URL = os.environ.get("INFRAI_BASE_URL", "https://api.example.test/v1")
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def call(method, path, payload=None, idempotency_key=None):
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(4):
        response = requests.request(
            method, BASE_URL + path, json=payload, headers=headers, timeout=15
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("Rate limit persisted after retries")


template = call(
    "POST",
    "/email/template/create",
    {
        "name": "password-reset",
        "subject": "Reset your account password",
        "html": "<p>Hello {{user_name}},</p><p><a href=\"{{reset_link}}\">Reset password</a></p>",
        "variables": ["user_name", "reset_link"],
    },
    idempotency_key="template-password-reset-v1",
)
template_id = template["id"]

preview = call(
    "POST",
    f"/email/template/preview/{template_id}",
    {"variables": {"user_name": "A. Customer", "reset_link": "https://app.example/reset/test"}},
)
if "{{" in preview.get("html", ""):
    raise ValueError("Preview still contains an unresolved template variable")

required = {"user_name", "reset_link"}
values = {"user_name": "A. Customer", "reset_link": "https://app.example/reset/abc"}
if set(values) != required:
    raise ValueError("Password reset variables do not match the template contract")

call(
    "POST",
    "/email/send",
    {
        "to": ["customer@example.com"],
        "template_id": template_id,
        "variables": values,
    },
    idempotency_key=f"password-reset-{uuid.uuid4()}",
)
```

The important check is the preview assertion. It turns “the email looked strange” into a deterministic test fixture. I once spent an afternoon chasing an API error that was really a one-character placeholder mismatch; logging the rendered preview would have ended that investigation in minutes. Short tests pay for themselves here.

Catch it early.

For a longer-running service, I would add a fixture matrix: one row for a normal name, one for a non-ASCII name, one for an expired link, and one with every required variable intentionally removed. The test should assert both the rendered HTML and the API response shape, then preserve the request ID in the test report. That extra work matters because a reset email crosses several boundaries—template interpolation, HTML escaping, URL construction, provider validation, and eventual delivery—and a green unit test that skips preview only verifies the easiest boundary. I don't treat a template update as ready until the same fixture passes after the update route is called.

## What changes between API templates and an SMTP relay?

An API-first service keeps template validation, send requests, and delivery status in one application boundary. An SMTP relay gives you a familiar mail protocol and often fits organizations that already operate a mail gateway, DKIM policy, queue, and archive. Neither option removes the need to validate reset links and variable names before sending.

| Option | Strength for password resets | Trade-off to accept |
| --- | --- | --- |
| Amazon SES | Mature sending controls and detailed AWS integration | You own template conventions and AWS configuration |
| SendGrid | Friendly template tooling and broad transactional-email workflows | Another vendor account, API surface, and billing stream |
| Mailgun | Clear event-oriented email operations and developer APIs | You still design the application-side placeholder tests |
| SMTP relay | Works with an existing mail gateway and compliance archive | More transport configuration; template errors remain your responsibility |
| A unified REST backend | One key and one bill across backend capabilities, plus a direct HTTP interface | No SMTP relay, so teams requiring SMTP-specific controls should choose a relay |

Infrai fits this workflow with one key and one bill across backend capabilities, plus a plain REST API over HTTP that a Python service can call without installing an SDK. That advantage is operational, not a promise of higher inbox placement. The service also exposes public discovery and consistent HTTP conventions.

## Where does this approach stop fitting?

The catch is that there is no SMTP relay in this capability group. If your compliance design requires routing through a controlled mail gateway, stick with SES, SendGrid, Mailgun, or your internal relay and keep the preview tests in your application. The email side also has no hosted OTP interface, no cancellation for scheduled email, and no webhook event push; event workflows are pull-based. Those are capability boundaries, not template fixes.

For a fintech reset flow, I would also keep the reset token short-lived, single-use, and opaque, while ensuring the preview fixture never contains a real customer token. Your mileage may vary on whether HTML snapshots belong in source control, but storing at least the variable contract and a sanitized preview makes regressions easy to spot.

Operationally, promote a template only after three checks pass: the preview contains no unresolved placeholders, the backend payload has exactly the required variable names, and a test send is accepted with a recorded request ID. Treat a 4xx response as input-validation evidence, preserve its body in structured logs, and fix the payload before retrying. If delivery reliability is the deciding axis, measure accepted sends and downstream events separately; a successful API call is not the same thing as an inbox arrival.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://sendgrid.com/en-us/solutions/email-api
- https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages
- https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API
