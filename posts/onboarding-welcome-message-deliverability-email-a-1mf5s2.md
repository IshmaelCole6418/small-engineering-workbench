# Onboarding Welcome Message Deliverability: Email API Evidence for Media Compliance Notices

A media publisher needs its onboarding welcome message and compliance notice to reach readers, with an email API delivery record that someone can explain later. Start with a transactional email API, a durable notice record, and a separate record of delivery events. The useful selection criterion is whether that evidence can be recovered and interpreted after the send. A successful request alone is too little.

TL;DR: For onboarding welcome message deliverability, choose an email API by its authentication support, suppression controls, bounce handling, and exportable delivery evidence. Configure SPF and DKIM for the actual sending domain, keep compliance notices separate from optional welcome content, and distinguish submission from receiving-server acceptance. No email API can turn a transport acknowledgment into proof that a person read a notice.

For a Python AI application builder, this is a satisfying notebook-to-prod boundary: experiment with wording in a notebook, then put approved notice text, deterministic recipient checks, and an eval harness around the production send. Keep model calls out of the delivery-event path. Prompt cost belongs in content development; it should not grow with duplicate delivery callbacks.

## Build the evidence record before connecting the sender

The data flow is short. An onboarding action creates a notice identifier and freezes the approved content version; a recipient check determines whether sending is allowed; the sender adapter submits the message and records its external identifier; verified callbacks add evidence to the notice record. An operator can then retrieve the notice, its recipient decision, and its transport history together. For this design, welcome copy and the compliance notice have separate purposes and separate identifiers, even if the same onboarding action triggers both.

Here is a runnable local example using the standard library. It models the evidence reducer after callback verification, with synthetic events for a fictional media notice. The deliberately small vocabulary is internal to this example, not an external API contract. Mapping a provider's documented events into it is an explicit adapter responsibility.

```python
import hashlib
import json


def reduce_evidence(notice, events):
    result = {
        "notice_id": notice["notice_id"],
        "content_sha256": hashlib.sha256(
            notice["body"].encode("utf-8")
        ).hexdigest(),
        "recipient": notice["recipient"],
        "message_id": notice["message_id"],
        "observed_kinds": [],
        "suppress_future": False,
        "needs_review": False,
    }
    seen = set()
    kinds = set()
    for event in events:
        if event["message_id"] != notice["message_id"]:
            raise ValueError("Event belongs to another message")
        identity = (event["source"], event["event_id"])
        if identity in seen:
            continue
        seen.add(identity)
        kind = event["kind"]
        if kind not in {
            "submitted", "server_accepted", "permanent_failure"
        }:
            raise ValueError("Unmapped event kind")
        kinds.add(kind)
    result["observed_kinds"] = sorted(kinds)
    result["suppress_future"] = "permanent_failure" in kinds
    result["needs_review"] = {
        "server_accepted", "permanent_failure"
    }.issubset(kinds)
    return result


notice = {
    "notice_id": "media-notice-104",
    "recipient": "reader@example.invalid",
    "message_id": "message-104",
    "body": "Your publisher account notice is available.",
}
events = [
    {
        "source": "mail-adapter",
        "event_id": "event-1",
        "message_id": "message-104",
        "kind": "submitted",
    },
    {
        "source": "mail-adapter",
        "event_id": "event-2",
        "message_id": "message-104",
        "kind": "permanent_failure",
    },
]
evidence = reduce_evidence(notice, events + [events[0]])
assert evidence == reduce_evidence(notice, list(reversed(events)))
assert evidence["observed_kinds"] == [
    "permanent_failure", "submitted"
]
assert evidence["suppress_future"]
assert not evidence["needs_review"]
print(json.dumps(evidence, indent=2))
```

The fixture contains 3 input entries but only 2 distinct event identifiers. Replaying the submission cannot erase the permanent failure, and reversing arrival order produces the same result. That is the narrow behavior the assertions evaluate.

The function does not send mail, authenticate callbacks, persist records, or implement a complete suppression service. Its output is a projection; retain original events separately.

That boundary matters. A content digest lets an auditor compare bytes against a retained copy, but the digest alone establishes neither when the notice existed nor who received it. Save the rendered notice, template version, recipient decision, timestamps, and external message mapping under an access and retention policy.

Avoid putting reader addresses or notice bodies into general application logs.

## How should an email API document onboarding welcome message deliverability?

Name the claim.

An API submission record says the sending service accepted a request under that service's documented contract. It does not establish receiving-server acceptance. SMTP acceptance, described in RFC 5321, concerns responsibility for a message at a particular transport hop; the identity of that hop matters. Even a receiving server's acceptance does not establish inbox placement or human reading. Delivery Status Notifications in RFC 3464 provide a machine-readable structure for delivery reports, but their meaning still depends on the reported action, recipient, and status. Preserve those distinctions when normalizing events. For the publisher, an auditable delivery record should therefore answer a bounded question: which approved notice was addressed to which recipient, what attempt was made, and what evidence followed? Whether that record satisfies a particular legal notice obligation requires the applicable rule and qualified review. The engineering record should expose its limits rather than label every accepted request as compliance complete. For example, keep the sending service's original status alongside the normalized category so that a later review can distinguish what the service actually reported from what the application inferred.

There is a concrete trap in a last-callback-wins status field. A delayed submission callback can arrive after a failure callback and make the dashboard look healthier while deleting the fact an investigator needs. The example keeps observed facts together and flags contradictory acceptance and failure evidence for review. In production, retain attempt identifiers as well: a successful retry must not silently overwrite the history of an earlier attempt.

## Select for authentication and recipient decisions

SPF, specified in RFC 7208, checks whether a sending host is authorized for the evaluated envelope identity. DKIM, specified in RFC 6376, lets a verifier check a domain's signature over selected message headers and body content. Neither is a promise of inbox delivery. Begin with these mechanisms because an onboarding welcome message should use an intentional sending identity, then inspect received-message authentication results in controlled mailbox tests. A DNS configuration screen is only one part of that check.

A beginner evaluating an email API can use the same fixture across candidates: one approved media notice, a controlled recipient, the intended sender identity, and an expected evidence record. Ask whether the service exposes a stable message identifier, documents callback authentication, distinguishes temporary from permanent delivery failures, and supports exporting events with enough detail to reconcile an attempt. Document the answers before integrating.

The best fit is the service whose documented boundaries let the team make its required evidence claim.

Suppression needs an equally explicit boundary. A permanent address failure and a reader opting out of optional welcome content are different reasons to stop a send. Store the reason and scope with the decision.

For this workflow, suppress further automatic attempts to a permanently failed address pending review, and record an exception for the notice owner to resolve through an approved process. Do not repeatedly submit to the same address merely because the notice is important.

RFC 8058 specifies one-click unsubscribe signaling for list email. Apply its semantics where that message class requires them; don't treat an unsubscribe mechanism as a universal rule for every account notice. Have the publisher's responsible team classify the compliance notice and optional welcome content before encoding recipient policy. Keeping the messages separate makes that policy easier to inspect, at the cost of maintaining two content and routing decisions. That is a useful trade-off when the audit question concerns one specific notice.

Changing channels doesn't settle it. If an SMS fallback is proposed, evaluate its consent, opt-out, and record requirements separately, using the CTIA messaging guidance as a starting point for the messaging review. An email failure is not evidence of permission to text someone.

## Take the notebook experiment through an operational rehearsal

An eval-driven rollout starts with evidence fixtures, not a polished welcome paragraph. Extend the local harness with a duplicate failure, an event for another message, an unknown event kind, and acceptance followed by failure. Test the callback verifier independently, including rejected signatures. Then test persistence and replay across a process restart; the in-memory set in the example has no durability claim.

Use a durable queue or equivalent persisted handoff around the send worker, and decide how ambiguous submission timeouts will be reconciled before enabling automatic retries. A timeout can leave the application without a confirmed submission result. Retrying without reconciliation can create another notice attempt; the evidence model needs to retain that possibility. Prefer an explicit pending investigation state when the sender contract cannot resolve it. This costs operator time, but it preserves an honest record.

Reconciliation is part of the send workflow. Give an unresolved attempt an owner instead of leaving it in a queue that nobody inspects.

The production rehearsal should show that an operator can retrieve the exact retained notice, explain its recipient decision, and trace every attempt to its verified events. It should also show alerts for old pending attempts, callback verification failures, and unmapped event kinds without exposing message content in alert payloads. Assign an owner to review unresolved records, agree on retention and access rules, and rehearse recovery from a stopped worker. Finish by replaying the fixtures against a fresh deployment.

Keep prompt-cost awareness in the right place: evaluate generated wording before approval, retain the approved result, and avoid regenerating it on transport retries. That keeps repeated attempts tied to the same content evidence.

The release decision is then concrete: ship when the team can explain both a successful delivery record and an unresolved one, with the same precision.

## References

The transport claims above use the SMTP and Delivery Status Notification specifications. The authentication discussion uses the SPF and DKIM specifications; the unsubscribe discussion uses the narrower one-click unsubscribe specification. These sources describe mechanisms. They do not determine whether a publisher has met a particular legal notice obligation.

## Further reading

- RFC 5321, Simple Mail Transfer Protocol: https://datatracker.ietf.org/doc/html/rfc5321
- RFC 3464, An Extensible Message Format for Delivery Status Notifications: https://datatracker.ietf.org/doc/html/rfc3464
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
- CTIA messaging interoperability and compliance best practices: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
