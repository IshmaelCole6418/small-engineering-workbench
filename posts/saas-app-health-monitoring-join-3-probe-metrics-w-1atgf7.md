# SaaS App Health Monitoring: Join 3 Probe Metrics With Pipeline Logs

Container recovery and pipeline completion answer different questions. For a marketplace's nightly data pipeline, use startup, readiness, and liveness probes to manage the container, then join their failure logs to the pipeline run before attributing investigation or model-call costs. TL;DR: expose app health endpoints, mirror failures into structured logs and counters, and keep a readiness gauge; use a separate heartbeat check to establish whether the scheduled job actually ran.

A responsive process can still miss tonight's catalog refresh. Don't infer completion from availability. That distinction matters more than a reassuring green dashboard.

This note uses a Python example for the notebook-to-prod path. The same probe contract applies to a Node.js service packaged in Docker and deployed on Kubernetes. The editorial angle is evidence lineage: preserve the run and stage that produced a health signal so a later search, dashboard, or AI incident summary can explain what it actually observed.

## How should startup, readiness, and liveness probes feed app health monitoring?

Startup answers whether initialization has completed. Readiness answers whether the app should receive traffic. Liveness answers whether the process needs recovery. Kubernetes uses these signals differently: a failed readiness probe removes a Pod from Service traffic, while repeated startup or liveness failures can trigger a container restart. A configured startup probe prevents readiness and liveness checks from running until startup succeeds.

The trade-off is explicit: app probes support container recovery, but they cannot prove that the nightly batch completed. Keep those meanings separate. A temporarily unavailable downstream catalog source should not automatically become a liveness failure, because restarting the worker may discard progress without fixing the source.

The data flow is small: app endpoints report health, each failed check emits a structured event and increments a counter, and the current readiness state feeds a gauge. An operator searches events by run and stage, then correlates timestamps and any available trace identifiers with pipeline evidence. A separate completion heartbeat covers the scheduled job. The probe log provides context for attribution; it does not establish that a billable operation happened.

For an AI builder, this also sets a useful prompt boundary. Retrieve the relevant run's evidence first. Sending an entire night's logs to an incident-summary model increases prompt volume and makes unrelated marketplace stages compete for attention.

## Run a small probe contract before choosing the backend

Save the following as a Python file and run it with Python 3. It starts a local HTTP server on port 8080. The paths and JSON fields are this example's own application contract; they are not a vendor ingestion schema.

Initialization takes an illustrative 2 seconds. The first event is an explicitly labeled synthetic failure so the example always has something to search, even if the first HTTP request arrives after initialization.

```python
import json
import threading
from collections import Counter
from datetime import datetime, timezone
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlsplit

initialized = threading.Event()
lock = threading.Lock()
failures = Counter()
events = []
context = {
    "pipeline": "marketplace_catalog",
    "run_id": "example-nightly-run",
    "stage": "catalog_load",
    "cost_center": "catalog_operations",
}


def record_failure(probe, synthetic=False):
    event = {
        **context,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "event": "probe_failure",
        "probe": probe,
        "synthetic": synthetic,
    }
    with lock:
        failures[probe] += 1
        events.append(event)
    print(json.dumps(event), flush=True)


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        path = urlsplit(self.path).path
        if path == "/metrics":
            with lock:
                payload = {
                    "probe_failures_total": dict(failures),
                    "readiness": int(initialized.is_set()),
                }
            status = 200
        elif path in {"/startup", "/ready", "/live"}:
            probe = path.removeprefix("/")
            healthy = probe == "live" or initialized.is_set()
            if not healthy:
                record_failure(probe)
            payload = {"probe": probe, "healthy": healthy}
            status = 200 if healthy else 503
        else:
            payload, status = {"error": "unknown application path"}, 404

        data = json.dumps(payload).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)


if __name__ == "__main__":
    record_failure("ready", synthetic=True)
    with lock:
        matches = [
            event for event in events
            if event["run_id"] == context["run_id"]
            and event["stage"] == "catalog_load"
        ]
    assert len(matches) == 1
    assert matches[0]["cost_center"] == "catalog_operations"
    print(json.dumps({"local_search_matches": matches}), flush=True)

    threading.Timer(2.0, initialized.set).start()
    HTTPServer(("127.0.0.1", 8080), Handler).serve_forever()
```

The code runs without an SDK or external service. Its metrics response is application JSON, not a Prometheus exposition endpoint. The local search filters Python dictionaries; it makes no assumption about a hosted search API's filter syntax.

There are deliberate boundaries here. The liveness endpoint proves that this server can answer a request; it does not detect a separate worker thread that has stopped making progress. Readiness follows initialization only. A production worker needs readiness and progress conditions tied to its actual work, plus a collector that persists events beyond process memory. Bind to an appropriate container interface before attaching Kubernetes probes; loopback binding here keeps the demonstration local.

Test the meanings.

An eval-driven rollout should exercise initialization, temporary dependency loss, a stalled worker, and an absent scheduled run. Assert the expected probe outcome for each fixture, and verify that the relevant failure event contains the same run identifier. The example's assertion checks local attribution only; it is not evidence of deployment reliability or an end-to-end monitoring test.

The next Python example is the read-only backend handoff. Configure INFRAI_API_ORIGIN with the service's HTTPS API origin, without a path, and INFRAI_API_KEY with your key. It searches logs without undocumented filter parameters and prints the returned JSON without assuming a response envelope. Apply run-level selection only after inspecting the documented response structure.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

origin = os.environ["INFRAI_API_ORIGIN"].rstrip("/")
if not origin.startswith("https://"):
    raise ValueError("The API origin must use HTTPS")
url = origin + "/v1/logs/search"

for attempt in range(4):
    request = Request(
        url,
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    try:
        with urlopen(request, timeout=20) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            result = json.load(response)
        print(json.dumps(result, indent=2))
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        delay = 2 ** attempt
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(
                    delay,
                    (retry_at - datetime.now(timezone.utc)).total_seconds(),
                )
        time.sleep(delay)
```

This request performs no write, so retrying it cannot duplicate an ingested event. The bounded retry loop honors Retry-After and surfaces an HTTP error body if the query fails. A production collector still needs an explicit ingestion contract and durable delivery; the probe server above does not perform that integration.

## Attribute evidence before attributing cost

The important join is from a health event to a run, then from that run to actual work records. Keep a stable run identifier across retries, and give distinct attempts their own identifiers in the production event design. Otherwise, a retry can make one logical batch look like several unrelated investigations, or make several attempts collapse into one misleading success.

Probe counts are not expense records.

A failure counter says how many checks failed during that process lifetime. Its value depends on probe frequency as well as service behavior, so a larger count alone cannot establish greater user impact or more model spending. The readiness gauge describes current state and loses the history that a log preserves. Use both, and inspect timestamps before assigning a pipeline stage responsibility for an incident.

Keep unbounded run identifiers in logs rather than copying each one into a separate metric series. For the dashboard, start with bounded dimensions such as service and probe kind, then use the structured events for the detailed run search. The example stores events in memory solely to make the contract inspectable. It also counts its synthetic event, so exclude that fixture from operational reporting when turning the demonstration into a service.

For RAG or agent features attached to the pipeline, retain actual call metadata alongside the run's work records. A probe failure should never stand in for a missing model invoice or a completed catalog update. Evaluate an incident-summary prompt against fixtures where health and completion disagree: a healthy server with no finished batch, and a finished batch followed by an unhealthy server. A useful summary must preserve that distinction.

## Compare tools by the evidence each one supplies

The choice is about coverage and integration work. None of the following replaces the application decision about what readiness means.

| Option | Useful role for this pipeline | Boundary to account for |
| --- | --- | --- |
| Prometheus | Collect probe-failure counters and readiness gauges for service-state queries. | Requires suitable metric exposition; a gauge does not provide the structured run log. |
| Grafana Loki | Search application logs and preserve the run-level evidence behind a health transition. | Logs need consistent fields and timestamps; completion must be established explicitly. |
| Healthchecks | Track scheduled-job signals and identify a missing nightly run. | A job heartbeat does not explain which container health condition failed. |
| Infrai | Combine logs and metrics within a broader backend contract spanning 295 routes across 20 modules under one key; per-call cost, vendor, and latency metadata can support attribution for associated calls. | Use it for a compact operational view; choose dedicated tooling when built-in alert delivery, heartbeat monitoring, or distributed trace exploration is required. |

The broader backend surface fits a team adding capabilities alongside an AI application without introducing a separate integration for every module. Consistent per-call metadata is the distinct supporting advantage for this scenario: it gives the run's work records something concrete to associate with cost. It does not automatically allocate every marketplace expense.

The limitation is scope: this is not a full replacement for a dedicated uptime platform. Choose Healthchecks for missed schedules and a dedicated tracing platform for span-tree investigation. A compact logs-and-metrics setup can't answer those questions on its own. Plan alert ownership explicitly, using a query consumer or a dedicated monitoring system where needed. Correlating timestamps and log fields such as trace_id and span_id can support investigation, but those fields do not constitute a distributed tracing query interface or a span tree.

Crash symbolication, source-map decoding, and Session Replay belong in a separate evaluation if the incident workflow requires them. Likewise, verify retention, export, subscription, and user-erasure requirements before storing identifiable marketplace data. This walkthrough establishes none of those lifecycle guarantees; keep sample events free of customer records.

A sensible starting combination is metrics for state, searchable logs for attribution, and a heartbeat tool for schedule coverage. Add components when an operational question needs their evidence. That keeps notebook-to-prod work focused without pretending that a small dashboard is a complete uptime platform.

## Put the contract through an operational review

Before deployment, connect each Kubernetes probe to the corresponding application endpoint and choose timing from the workload's initialization and recovery behavior. Test that readiness failure stops traffic without turning dependency trouble into an unnecessary restart. Also test the startup gate: an application still loading its initial state should not face a premature liveness decision.

Persist the logs, check the counter reset behavior after restart, and confirm that a failing probe can be found under the intended run. Keep the demonstration's synthetic event out of real dashboards. Exercise the missing-run case through the independent heartbeat path, because no failed probe may exist when the job never starts.

Then review the attribution output with the same discipline as an eval result. Every assigned cost should point to a work record; every health claim should point to a state observation. If an AI summary cannot produce that evidence, leave the claim unassigned. That is a useful stopping rule for a small marketplace team.

## Sources

References for probe semantics and the complementary monitoring roles discussed above:

- Kubernetes, Configure Liveness, Readiness and Startup Probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Prometheus, Metric Types: https://prometheus.io/docs/concepts/metric_types/
- Prometheus, Instrumentation Best Practices: https://prometheus.io/docs/practices/instrumentation/
- Grafana Loki documentation: https://grafana.com/docs/loki/latest/
- Healthchecks documentation: https://healthchecks.io/docs/
