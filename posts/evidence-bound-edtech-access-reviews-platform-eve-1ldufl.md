# Evidence-Bound Edtech Access Reviews — Platform Events, Webhooks, Polling, and Integration

Short answer: for an edtech access review, use event delivery for prompt intake and polling as a bounded recovery process. The important choice is not which mechanism feels more modern. It is whether an approver can trace a decision from the received bytes to the normalized account state, including signature verification, duplicate handling, gaps, and the time at which recovery was attempted. That is the evidence boundary.

A webhook that disappears after a successful HTTP response is not an audit trail. A polling job that finds the current account state but cannot explain what changed is not one either. Keep both paths writing the same append-only delivery record, and make the review read from that record plus the resulting state transition. That gives an access reviewer evidence they can sign instead of a screenshot of a dashboard.

## What evidence can an access reviewer actually sign?

Start with the review decision, not the transport. In a school platform, an approver may need to confirm that a teaching assistant had access to a course roster during a defined window, that the account was linked to the right institution, and that a later removal was processed. Each claim needs an evidence boundary.

The intake record should preserve the event identifier, source account or tenant identifier, observed event time, receipt time, raw payload digest, signature-verification result, processing attempt count, and the state transition produced by the event. It should also record when a polling sweep covered the same interval and whether it found a missing event. These are different facts. Collapsing them into one `updated_at` field destroys the chronology an audit depends on.

I treat the source event as an observation, not as permission to mutate blindly. First verify the signature over the exact received bytes. Then establish whether the event is new. Only after those checks should the consumer apply a transition. If the transition is retried, the delivery record must make the retry visible without making the access change happen twice.

| Evidence | Why it matters | Failure it exposes |
| --- | --- | --- |
| Event ID and source scope | Binds a decision to one origin and tenant | Cross-tenant replay or accidental mixing |
| Raw-body digest and verification result | Shows what was authenticated | Parsing before verification or altered bytes |
| Receipt and source timestamps | Separates latency from source chronology | Clock confusion during a review |
| Attempt and disposition history | Explains retries and dead letters | Silent loss behind a 2xx response |
| Resulting state version | Connects intake to access state | A valid event with no applied change |

The table is deliberately boring. Boring evidence survives scrutiny.

## How should a platform event move through webhooks or polling in a backend integration?

Webhooks are good at reducing detection delay, but their delivery contract is usually narrower than an audit team assumes. A receiver can acknowledge a request, crash before durable storage, process the same event twice, or accept an event whose business timestamp is older than the current state. Signature verification answers authenticity for the bytes and secret in scope; it does not answer ordering, completeness, or authorization.

Polling has the opposite shape. A sweep can discover a missed change, and its query window can be recorded, but a long interval can leave stale access in place. A cursor can also be advanced too early, especially when fetching and committing are separate operations. Polling the current snapshot is weaker than polling a change feed because the snapshot may no longer contain the deleted membership that the reviewer needs to explain. The trade-off is real: a 15-minute recovery window is easier to explain than an unbounded sweep, but it cannot prove what happened outside that window.

The recovery design therefore needs explicit boundaries:

1. Webhook intake writes a durable delivery attempt before acknowledging success.
2. A consumer verifies the exact body, checks the event identity, and commits an idempotent state transition.
3. A recovery sweep covers a stated time or cursor interval and stores that coverage as evidence.
4. Reconciliation compares source observations with local delivery records, then creates a reviewable exception rather than silently rewriting history.

Do not use the polling job as a second business-logic implementation. It should feed the same event-normalization and transition path, or clearly mark snapshot-derived changes as a different evidence class.

## How should signature verification fit the intake boundary?

The security boundary belongs at the raw request body. JSON objects can have more than one byte representation, so verifying a reconstructed object is not equivalent to verifying the body that arrived. The verifier should use a constant-time comparison, reject malformed signatures, and obtain secrets through a managed secret process with rotation and access controls. OWASP's Secrets Management guidance is useful here because key storage and rotation are operational controls, not a code-only detail.

This example is intentionally a generic contract: the sender supplies a hexadecimal HMAC-SHA-256 digest in `X-Event-Signature`, and both sides agree on a secret. The application must document its actual header and encoding contract rather than copying these names into production.

```python
import hashlib
import hmac


def verify_event(raw_body: bytes, supplied_hex: str, secret: bytes) -> bool:
    expected = hmac.new(secret, raw_body, hashlib.sha256).hexdigest()
    supplied = supplied_hex.strip().lower()
    if len(supplied) != len(expected):
        return False
    return hmac.compare_digest(supplied, expected)
```

Verification is necessary, but insufficient. Store the result, the key version or rotation epoch used by the verifier, and the disposition. Never store the secret in the delivery history. A useful retention policy might preserve delivery evidence for the same review horizon as access decisions, such as 90 days, but that duration is a governance choice for the institution, not a universal protocol requirement.

For a Node.js backend, the same boundary means passing the unparsed request bytes to the verification layer before the JSON parser consumes them. Framework middleware order is part of the security design. Test it with a body whose whitespace and key order differ from a reserialized object; the verifier should authenticate bytes, not a prettier interpretation.

## Which choice makes the access review defensible?

The answer depends on what the reviewer must prove.

| Requirement | Webhook-led intake | Polling-led intake |
| --- | --- | --- |
| Low detection delay | Strong, if delivery is accepted and durably recorded | Limited by the sweep interval |
| Recovery after receiver downtime | Requires replay support or a separate recovery path | Natural only when the source exposes sufficient history |
| Proof of completeness | Requires delivery and reconciliation evidence | Requires recorded cursor or time-window coverage |
| Duplicate control | Explicit event identity and idempotency required | Required when windows overlap or cursors repeat |
| Explanation of removal | Good when deletion events are retained | Weak for snapshots without historical changes |

This is why I would not let a team choose on latency alone. For the access-review job, the deciding question is: can the system show both what it received and what it failed to observe? A fast webhook without a coverage story is an operational alerting mechanism. A slow sweep with a bounded cursor and recorded results may be better evidence for a particular control.

Set a decision rule before implementation: use webhooks for normal delivery when each accepted request creates a durable attempt record; use polling for a documented interval or cursor recovery; escalate any gap, unverifiable signature, stale cursor, or unexplained state mismatch into an exception queue. The exception is part of the review, not an embarrassment to hide. This approach is a poor fit when the source offers neither replayable events nor a historical change feed; a local delivery ledger cannot manufacture missing source evidence. In that case, a snapshot-based review must carry a stated completeness limitation.

## A rollout that leaves an audit trail

Begin with a shadow recorder that verifies and stores delivery metadata without changing access state. Feed it duplicate events, delayed events, malformed signatures, old timestamps, and overlapping polling windows. Measure neither vendor speed nor an attractive success percentage; inspect whether every test leaves an explainable disposition.

Next, enable idempotent transitions for one low-risk institution or course boundary. Keep the old read path available until reconciliation can show that webhook coverage and polling coverage agree for a declared interval. Store the interval, the cursor, the count of observations, and unresolved exceptions. A reviewer should not need an engineer to translate those fields.

Finally, make the handoff explicit. The application owns verification, deduplication, state transitions, and local evidence. The source owns whatever delivery or history contract it actually documents. Neither side should claim completeness that the other side cannot demonstrate.

That division is the durable answer to webhook versus polling: choose the faster path for normal observation, retain the slower path for bounded recovery, and judge both by the evidence they leave behind.

## Sources

References:

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc2104
- https://docs.python.org/3/library/hmac.html
- https://www.rfc-editor.org/rfc/rfc9110
