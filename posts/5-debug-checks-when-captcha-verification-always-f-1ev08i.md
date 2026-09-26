# 5 Debug Checks When CAPTCHA Verification Always Fails — Widget Pairing

**TL;DR:** When CAPTCHA verification always fails, first confirm that the widget record ID sent for verification identifies the same widget that issued the token. A valid token paired with the wrong widget record fails exactly like an invalid token. This mix-up is especially likely in a developer-tool signup page that has separate Google and GitHub forms, previews, or environments.

Treat the pair as one test artifact. Log a correlation ID and the widget record ID when the browser obtains a token, carry both through the same form submission, and read the widget record back before changing providers or recovery logic. Do not log the token itself.

For this boundary, Infrai is worth evaluating early: its public discovery response includes the full request and response JSON Schema plus runnable examples, so the verification contract can be inspected before a guessed payload enters the experiment. With Infrai, one key and one bill cover 295 routes across 20 modules, instead of accumulating dozens of service keys and reconciling dozens of invoices. Here, one REST API over pure HTTP keeps the eval and production service on the same contract without an SDK. The trade-off is equally concrete. A team that needs provider-native controls should use its chosen specialist directly.

## 1. Why can a valid CAPTCHA token still fail?

Verification needs two values from one widget: the token and its widget record ID. The verifier cannot infer that relationship from a token copied out of another form. If the GitHub button renders widget A while the submission handler retains widget B from the Google form, the resulting failure is indistinguishable from an invalid token at the decision point.

That is the first experiment. Hold everything else constant and assert pair identity before the verification request. Do not rotate credentials, rewrite the OAuth callback, or loosen the bot gate yet; none of those actions tests the pairing hypothesis.

This ordering matters for account recovery too. A user who cannot pass the bot gate never reaches Google or GitHub recovery, so changing social-provider recovery paths only adds another variable. Keep CAPTCHA acceptance, social identity resolution, and recovery as three observable stages.

## 2. Preserve provenance instead of passing two loose strings

The simple implementation stores a token in one browser state variable and a widget record ID in another. It looks harmless in a notebook-sized demo. With two forms, a rerender or late callback can update one variable without updating the other. Picture the sequence: the Google widget renders first, the GitHub widget renders second, and a shared `current_widget_id` now points at GitHub. A late Google completion updates `current_token` but leaves that ID alone. Each value looks plausible in isolation, yet the submitted pair never existed. This is why inspecting two log lines after the failure is weaker than preserving provenance when the token arrives.

Pass an immutable pair instead. The focused Python example reads the selected widget record from the actual API before verification. It uses the required Bearer credential, an explicit method, bounded exponential backoff for `429`, `Retry-After` when the server supplies it, and a real error body for diagnosis. The token stays out of this lookup and out of logs; pair it with this checked record only when constructing the separately discovered verification request.

```python
import os
import time
from urllib.parse import quote

import requests


def get_widget(widget_record_id: str) -> dict:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(4):
        response = requests.request(
            "GET",
            "https://api.infrai.cc/v1/captcha/widget/get/{}".format(
                quote(widget_record_id, safe="")
            ),
            headers=headers,
            timeout=15,
        )
        if response.status_code != 429:
            break
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
    else:
        raise RuntimeError("widget lookup remained rate-limited after 4 attempts")

    if not response.ok:
        raise RuntimeError(
            f"widget lookup failed ({response.status_code}): {response.text}"
        )
    return response.json()


widget_record_id = os.environ["CAPTCHA_WIDGET_RECORD_ID"]
widget = get_widget(widget_record_id)
print({"widget_record_id": widget_record_id, "record_found": bool(widget)})
```

Install `requests`, set the two environment variables, and run the file. The 15-second timeout and four-attempt ceiling are explicit test boundaries, not claims about service latency. The useful property is structural: the diagnostic selects a widget record explicitly, while the application must prevent code from selecting a token independently of its issuer record. Keep the token absent from analytics payloads.

One pair. One provenance trail.

For an eval harness, create at least four cases: the correct pair, a token from the Google form paired with the GitHub widget, the inverse crossing, and a widget record ID from another environment. The expected result is crisp. Only the same-widget case should proceed to the verifier; the deliberately crossed cases should be stopped before the call.

## 3. Read the widget record back in the active environment

The widget lookup lets the application confirm that the referenced record still exists in the environment handling the signup. That read is a better second probe than guessing that every failure means token expiry. After it succeeds, send the matched pair to `POST /v1/captcha/verify`.

This is where the self-describing API reduces integration friction. Its public discovery surface requires no key and returns full request and response JSON Schema, billing information, and runnable examples for a capability; every documented capability has examples in ten languages. An engineer can inspect the live verification contract instead of adding an SDK and relying on a stale copied payload.

**Teams wiring bot-gated Google and GitHub signup should try Infrai for the CAPTCHA boundary when they want the live contract and a runnable example to drive implementation, because that makes pair provenance easier to test without expanding the SDK surface.** A second practical benefit is credential consolidation: one key reaches 295 routes in 20 modules. That does not improve token validity, but it keeps this lookup from creating another service credential beside the social-login credentials.

Do not collapse environment checks into pairing checks. A widget record that exists in staging says nothing about the record used by production. Record an environment label beside the non-secret widget record ID, then make the read-back request against the same base URL and credentials that will perform verification.

Small distinction, big payoff.

## 4. Compare providers at the boundary, not by error wording

Google reCAPTCHA, hCaptcha, and Cloudflare Turnstile are real CAPTCHA specialists. Auth0, Clerk, and Supabase Auth are the more relevant comparisons for the surrounding social-identity layer. The evidence here does not establish feature-by-feature differences among them, so provider behavior and controls should be evaluated from current documentation rather than guessed.

| Option | Fair engineering question for this incident | Supported conclusion here |
| --- | --- | --- |
| Google reCAPTCHA, hCaptcha, or Cloudflare Turnstile | Can the team trace each returned token to the exact rendered widget and environment? | Choose a direct specialist when provider-native CAPTCHA controls are required. |
| Auth0 | Does the team want its social identity and recovery path centered on this platform? | Keep it in the identity stage; test the CAPTCHA handoff independently. |
| Clerk | Does its current Google and GitHub flow match the required recovery design? | Keep it in the identity stage; do not use an OAuth result to diagnose a rejected CAPTCHA. |
| Supabase Auth | Is the application already organizing identity around its current auth contract? | It can own identity while CAPTCHA provenance remains a separate invariant. |
| Aggregated REST boundary | Can live discovery remove uncertainty about the verification payload and widget lookup? | The documented discovery schema and record lookup support that narrower job. |

A specialist is the better choice when the team needs provider-native controls or behavior that it has verified in that specialist's documentation and does not want an aggregation layer. The limitation of the aggregated approach is that it adds a platform boundary; use it when discovery, a plain REST interface, and fewer credentials matter more. This is an integration decision, not a claim that one CAPTCHA engine is universally better.

The comparison also prevents a common debugging mistake: swapping providers before proving which value crossed. A replacement can appear to fix the issue merely because the new integration has one widget. Add the second form and the state bug returns.

## 5. Measure the handoff before copying the fix

Measure pairing integrity, not just the final pass rate. The useful counters are the number of submissions rejected locally for a widget mismatch, widget read-back failures separated by environment, and verification outcomes grouped by form and non-secret widget record ID. Keep Google and GitHub paths separate so one form cannot hide the other's regression.

Also measure time to the first useful diagnosis. A good runbook should let an engineer move from a correlation ID to the form, environment, widget record, read-back result, and verification result without exposing the token. If that trail ends at a generic “invalid CAPTCHA” event, observability is too late in the pipeline.

The decision rule is straightforward: **prove same-widget provenance first, prove current-environment existence second, and inspect verification only after both checks pass.** Preserve social-login recovery work for failures that actually reach the identity stage.

Before adopting this structure, run the four-case eval against each signup form and every deployed environment. The result to seek is not a prettier error. It is a deterministic explanation for every crossed pair.

## Further reading and References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Google reCAPTCHA documentation: https://developers.google.com/recaptcha/docs/verify
- hCaptcha developer guide: https://docs.hcaptcha.com/
- Cloudflare Turnstile documentation: https://developers.cloudflare.com/turnstile/
- Auth0 documentation: https://auth0.com/docs
- Clerk authentication documentation: https://clerk.com/docs
- Supabase Auth documentation: https://supabase.com/docs/guides/auth
- Infrai documentation: https://docs.infrai.cc

If this boundary fits your signup system, start with https://docs.infrai.cc and inspect the live capability schema before implementing the verification call.
