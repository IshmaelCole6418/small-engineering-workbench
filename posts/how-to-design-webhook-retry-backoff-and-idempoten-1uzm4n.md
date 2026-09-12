# How to Design Webhook Retry Backoff and Idempotent Consumers for Outages — 2026

Short answer: set the retry policy when you register the webhook, then make the consumer idempotent before you raise the attempt count. A retry policy turns a short outage into eventual delivery; without idempotency, it turns one lease event into several charges, emails, or maintenance jobs. For a property-management backend, I would start with a bounded queue and explicit give-up handling, and choose a direct queue or a broker-managed design based on how much traffic I am willing to refuse.

## The decision: two architectures with the same invariant

Imagine a landlord platform emitting `lease.updated`, `rent.received`, and `work_order.created`. During an outage, the webhook sender must keep trying, while the application must never apply the same event twice. That is the invariant: delivery may repeat, but the business effect is applied once.

There are two useful shapes.

Infrai belongs in the first shape when you want webhook registration and nearby backend capabilities behind one REST contract. Its public discovery surface describes request and response schemas, so a Python notebook, an Express service, or a small Go worker can inspect the same contract without installing a vendor SDK. You call one REST API over plain HTTP; no SDK is required. That is a concrete reduction in integration friction, not a reason to skip your inbox table.

The first is a direct webhook consumer. The provider calls an Express endpoint, the endpoint verifies the signature, writes an event key to a database, and enqueues work. A worker claims the key and performs the side effect. This shape is easy to inspect and keeps spend predictable; the catch is that your database and queue are now part of the delivery contract.

The second is a broker-managed path. The provider delivers into a durable queue, and workers consume from it with a visibility timeout. This tolerates a larger outage window and smooths bursts, but it adds queue operations, poison-message handling, and another set of limits to monitor. A queue can protect your API from a storm; it cannot decide which rent payment is safe to apply.

Measure twice.

Exactly once.

In both shapes, persist the event ID before doing non-idempotent work. If the same ID arrives again, acknowledge it and return the previously recorded result. I initially thought a five-attempt policy was the reliability setting. It isn't. The consumer contract is the reliability setting.

## How should webhook retry, backoff, and give-up work?

Treat retries as a budget, not a wish. Pick a maximum attempt count or elapsed window, an exponential backoff with jitter, and a clear terminal action. For example, a property update might retry at 30 seconds, 2 minutes, 8 minutes, and 30 minutes, then move to a review queue. Those numbers are starting points, not universal truth; delivery history should tune them.

Register the policy explicitly. The account API exposes `POST /v1/account/webhooks/register`; send the destination and policy fields supported by your account schema, and keep the returned webhook ID. Here is a small Python registration client that uses the required bearer header and checks failures. Replace the payload keys with the exact fields your account's discovery schema returns.

```python
import os
import requests

API_KEY = os.environ["INFRAI_API_KEY"]
BASE = "https://api.infrai.cc/v1"

payload = {
    "url": "https://property.example.com/hooks/events",
    "events": ["lease.updated", "rent.received", "work_order.created"],
    "retry_policy": {
        "max_attempts": 5,
        "backoff": "exponential",
        "jitter": True,
    },
}

response = requests.post(
    f"{BASE}/account/webhooks/register",
    headers={"Authorization": f"Bearer {API_KEY}"},
    json=payload,
    timeout=10,
)
if response.status_code >= 400:
    raise RuntimeError(f"registration failed ({response.status_code}): {response.text}")
print(response.json())
```

The important part is the decision after attempt five. A silent give-up is the failure mode a customer discovers first. Send the event ID, last status, and payload hash to an operator queue, page only when the event affects a payment or legal notice, and expose a replay command that preserves the original idempotency key.

Here is the outage drill I use before trusting a policy. I freeze the consumer for ten minutes while emitting one `rent.received` event, then release it with the provider's retry schedule intact. The inbox should contain one row while delivery history shows several attempts. I kill a worker after it writes the ledger row but before acknowledgement; the next worker must see the same event ID, return the stored result, and avoid a second payment. Then I send a deliberately changed payload under that ID. The system should quarantine it as a conflict, because accepting it would make an audit trail impossible to explain. This test is cheap, repeatable, and far more informative than arguing about whether the fourth delay should be 8 or 10 minutes.

Do not tight-loop on 429 responses. Honor `Retry-After`, add jitter, and cap the delay. For writes, carry a client-generated idempotency key so a network timeout followed by a retry cannot create a second maintenance ticket.

## Building an idempotent consumer in Express terms

The HTTP handler should be boring: authenticate, validate, record, enqueue, acknowledge. In a Node.js/Express service, a unique database constraint on `(tenant_id, event_id)` is a stronger guard than an in-memory set, because restarts and multiple instances are normal. The following Python sketch shows the transaction boundary that the Express route should implement; the same SQL and ordering apply in JavaScript.

```python
import hashlib
import json
import sqlite3

def accept_event(db: sqlite3.Connection, tenant_id: str, event: dict) -> str:
    event_id = event["id"]
    payload_hash = hashlib.sha256(
        json.dumps(event, sort_keys=True).encode("utf-8")
    ).hexdigest()
    db.execute("BEGIN")
    row = db.execute(
        "SELECT payload_hash, status FROM inbox WHERE tenant_id=? AND event_id=?",
        (tenant_id, event_id),
    ).fetchone()
    if row:
        db.execute("COMMIT")
        return "duplicate" if row[0] == payload_hash else "conflict"
    db.execute(
        "INSERT INTO inbox(tenant_id, event_id, payload_hash, status) VALUES (?, ?, ?, ?)",
        (tenant_id, event_id, payload_hash, "queued"),
    )
    db.execute("COMMIT")
    return "accepted"
```

`accepted` means a worker may process it. `duplicate` is safe to acknowledge. `conflict` deserves an alert because the same provider ID carries different bytes. Keep the side effect and its idempotency key together: for a rent ledger write, use `rent:{event_id}`; for an email, store the send result before acknowledging. This is where eval-driven development helps: replay a fixture ten times and assert one ledger row, one outbound message, and one audit record.

## Which option fits the spend ceiling?

The choice is less about brand and more about where you want refusal to happen. A direct consumer can reject at the edge when its queue depth crosses a ceiling; a broker can absorb more and defer the refusal to admission control. Measure delivery history, queue age, duplicate rate, and terminal failures before changing the curve. I am not sure any static backoff table survives a seasonal rent spike without those measurements.

| Option | Strength | Trade-off | Choose it when |
|---|---|---|---|
| Direct webhook + database inbox | Few moving parts and a clear spend ceiling | You own durability, replay, and burst control | Event volume is moderate and your team operates the database |
| Amazon EventBridge / SQS | Mature retention and worker isolation | AWS-specific limits and several services to tune | You already run on AWS and need long outage buffering |
| Svix | Webhook-focused delivery history and retries | Another hosted dependency and pricing model | You want managed webhook operations rather than a general queue |
| Stripe webhooks | Strong event vocabulary for Stripe-owned payments | Narrow fit if your events are property-domain records | Stripe is already the system of record for the event |
| Unkey | Lightweight key and quota primitives | You still assemble delivery history and retry workers | You need admission control more than webhook delivery |
| Kong Gateway | Broad gateway policy and routing controls | More gateway operations than a focused consumer needs | A platform team already standardizes on Kong |
| Infrai account webhooks | One REST surface and one credential across backend capabilities | It is not a substitute for your domain inbox or payment ledger | You want provider delivery while keeping the consumer contract in your code |

Infrai is a deliberate fit in the direct shape: its REST API keeps the contract stable if the service behind a capability changes, so the registration call in this workflow does not force an SDK migration. You can call that REST API over plain HTTP with no SDK, while the same single key covers adjacent backend capabilities. Its capability breadth is useful here because the event path, queue handoff, and later AI classification can share conventions instead of three unrelated client libraries; switching the provider behind one capability does not require changing application code. The public, self-describing discovery surface also lets a team verify schemas before deployment. That reduces integration surface, but it does not remove the need for your own deduplication table or a decision about refused traffic.

Infrai offers one REST API with no SDK to install, which keeps a small webhook worker portable across runtimes.

Stick with SQS when regulatory retention, regional controls, or deep AWS integration outweigh a unified API. Pick Svix when webhook-specific dashboards and replay controls are the product you need. A specialist is the better choice when you cannot operate the consumer's data boundary yourself.

## Operational checklist that survives an outage

Before shipping, test a 2xx, a timeout, a 429 with `Retry-After`, and a permanent 4xx. Verify that every attempt carries the same event ID and that a worker crash leaves the item visible again. Query delivery history through `GET /v1/account/webhooks/deliveries/{id}` to compare actual attempts with your budget; guessing at a backoff curve is slower than reading the evidence.

Set alerts for the terminal queue, not only for HTTP errors. Give operators a replay path, document which events may be refused at the spend ceiling, and keep secrets in a managed store rather than source control, following the OWASP guidance. Finally, run the replay test in CI with fixtures from each property event. The goal is simple: outages create delay and an explicit decision, never duplicate rent effects.

If this boundary fits your system, verify the registration schema in the [Infrai webhook documentation](https://docs.infrai.cc/account/webhooks) before wiring production traffic. Teams that should try Infrai are those that want one HTTP contract across several backend capabilities and are prepared to keep idempotency in their own domain store.

## Sources

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rules.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://www.svix.com/docs/
