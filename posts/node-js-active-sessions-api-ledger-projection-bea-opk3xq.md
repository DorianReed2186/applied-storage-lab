# Node.js Active Sessions API: Ledger Projection Beats Direct Device Reads

Use a durable session ledger projected into a user-facing inventory, rather than reading the live session store directly. For a media service scoring login risk from device fingerprints, the deciding constraint is account recovery: a newly recovered account needs one coherent place to review and revoke old access, even when the operational session store has expired, rotated, or partitioned records. **Choose the ledger projection when recovery must invalidate access across devices; choose direct reads only for a small, single-store system whose session list is explicitly best-effort.**

TL;DR: let the authentication service remain authoritative for whether a credential is accepted, but emit opaque session lifecycle records to a durable ledger and build the inventory from that ledger. Never expose bearer tokens or treat a fingerprint as proof of identity. The inventory is a control surface, not an audit log and not a second authentication service.

## Which invariants survive account recovery?

The first invariant is blunt: possession of a session-list response must not grant possession of a session. Return an opaque management identifier, creation and last-seen times, coarse device presentation data, and a status; keep refresh tokens, raw cookies, and full fingerprint material out of the response. OWASP recommends renewing the session identifier after authentication and other privilege changes, and it treats recovery mechanisms as alternate authentication paths that must not be weaker than normal authentication. Those two concerns meet here.

The second invariant is that recovery establishes a boundary. Once recovery succeeds, policy must say whether every earlier session is revoked, only sessions marked risky are revoked, or the user is offered an explicit choice. For a media account with creator access, payment settings, and ordinary playback devices, I would default to revoking all pre-recovery sessions and then issue a new session. That costs convenience on televisions and consoles, but it avoids asking a possibly compromised browser to distinguish friendly devices from the attacker's device.

Short identifiers matter. Store a random, opaque public ID separately from the secret session credential, and scope every inventory lookup and revocation by the authenticated account ID. A device label such as `Chrome on macOS` is presentation, derived from mutable input; it must never become an authorization key.

One subtle failure boundary is delayed activity. A player may report progress after its session has been revoked, and a risk pipeline may finish scoring an older login after recovery. Those events may update analytical records, but they must not reactivate the session projection.

Recovery wins.

## Decision record: ledger projection versus direct reads

The comparison is narrower than “database versus cache.” It asks whether the data structure optimized for credential validation should also carry user-visible history and recovery semantics.

| Decision axis | Durable ledger plus projection | Direct reads from session store |
|---|---|---|
| Recovery boundary | Records a monotonic recovery epoch and rejects older lifecycle updates | Requires every session record and replica to receive the recovery change |
| User inventory | Can retain revoked entries for a defined display window | Usually shows only records still present in the operational store |
| Revocation latency | Depends on a separate command reaching the authentication check path | Can be immediate when validation and mutation share one authoritative store |
| Partition behavior | Inventory can be stale while authentication remains conservative | Inventory availability follows the session store |
| Operational burden | Event schema, idempotency, projection lag, and reconciliation | Fewer moving parts, but history and cross-store recovery are harder |
| Valid scope | Multiple credential stores, device classes, or asynchronous risk scoring | One authoritative store and a best-effort inventory |

The ledger does not magically revoke anything. The revocation command still has to reach the component that validates sessions, and that component must fail according to an explicit policy if its revocation state is unavailable. This is the common design trap: teams make the read model durable, see a reassuring “revoked” badge, and forget that a cached credential can still be accepted elsewhere. Consider the ordering rather than the badge: recovery advances the account epoch, a projector marks five listed devices revoked, and then an activity event created before recovery arrives late. If the projector compares only event arrival times, that old activity can paint one device active again; if the authentication authority checks the epoch, the same stale event is harmless. The inventory needs ordering guards, while credential acceptance needs the current authoritative epoch. Neither can substitute for the other.

For a concrete data model, keep `session_public_id`, `account_id`, `credential_version`, `created_at`, `last_seen_at`, `revoked_at`, `recovery_epoch`, a coarse device label, and a risk state. Fingerprint observations belong in a restricted risk dataset keyed through a rotating pseudonymous handle, not in the inventory row. Browser entropy changes; shared televisions are real; copied fingerprints are possible. Treat the score as evidence for step-up or review, never as a stable device identity.

## Critical path and failure handling

The critical path has two writes with different purposes. First, recovery advances an account-level epoch in the authentication authority. Second, it appends a recovery event for projection and reconciliation. A session is acceptable only when its embedded or stored epoch equals the current epoch and it has not been individually revoked.

The following Python expresses the contract even if the surrounding service is Node.js. The transaction and outbox must commit together; a worker may publish the same event more than once, so the projector uses `event_id` for idempotency.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol
from uuid import UUID


class RecoveryStore(Protocol):
    def advance_epoch_and_enqueue(
        self, account_id: UUID, occurred_at: datetime
    ) -> tuple[int, UUID]: ...


@dataclass(frozen=True)
class RecoveryResult:
    recovery_epoch: int
    event_id: UUID
    occurred_at: datetime


def complete_recovery(account_id: UUID, store: RecoveryStore) -> RecoveryResult:
    occurred_at = datetime.now(timezone.utc)
    epoch, event_id = store.advance_epoch_and_enqueue(account_id, occurred_at)
    return RecoveryResult(epoch, event_id, occurred_at)


def session_is_acceptable(
    session_epoch: int,
    current_epoch: int,
    individually_revoked: bool,
) -> bool:
    return session_epoch == current_epoch and not individually_revoked
```

Three failure modes deserve explicit tests. Duplicate events must leave the projection unchanged. An older `last_seen` event arriving after a recovery event must not clear revocation or move the row into an active state. Finally, a projection outage must not block login validation or recovery; it may make the inventory temporarily stale, which the API should signal with an `as_of` timestamp rather than pretending the view is current.

Test reordered sequences, not just happy-path requests. Generate a login, activity, recovery, late activity, and duplicate recovery; apply every meaningful permutation; then assert that no session from the older epoch becomes active. Add an integration test that commits the epoch change while event publication is unavailable, restores the publisher, and verifies eventual projection. A gauge for oldest unpublished outbox record and a histogram for projection delay are more useful than a generic request-success dashboard.

## What Should an Active Sessions API Show Users About Their Devices?

Return enough information for recognition and action: an opaque management ID, coarse client and platform labels, approximate region when policy and consent permit it, creation time, last activity time, current-device status, and revocation status. Attach an `as_of` value to the collection. Avoid exact IP history in the ordinary response; it creates privacy and retention obligations while offering a false sense of device certainty.

Revocation should be idempotent. A repeated request for the same management ID returns the same effective state, while an ID owned by another account reveals nothing useful. OWASP's guidance on generic authentication and recovery responses is relevant here: response content and timing should not become an account-enumeration channel.

Risk scoring stays adjacent to this API, not inside its authority model. A sharp fingerprint change, impossible travel signal, or unfamiliar client can prompt reauthentication before showing or revoking sessions, but the score itself should be versioned and explainable enough for operators to diagnose false positives. Do not let a score of 37 versus 38 decide whether recovery invalidates old credentials. Recovery is a policy boundary.

Keep that line sharp.

## Why reject direct session-store reads?

Direct reads are rejected for this media system because multiple device classes and asynchronous fingerprint scoring create late updates, while account recovery requires a durable, ordered boundary. An operational store optimized for rapid credential checks is a poor historical interface: expiry can erase the record before a user reviews it, replication can produce contradictory lists, and a cache key often lacks the metadata needed to explain a device.

The ledger approach has real limitations. It is not suitable when the team cannot operate idempotent event delivery, monitor projection delay, and reconcile the view against the authentication authority; under those conditions its extra state can make recovery harder to reason about. The trade-off is a stale user-facing view in exchange for durable lifecycle history, and that is acceptable only because the view never decides credential validity. A system that promises immediate read-after-revoke display from every region needs either synchronous coordination or a different consistency design. Do not disguise that requirement with eventual-consistency language.

The rejected option still has a valid use case. For an internal application with one authoritative relational store, short-lived sessions, no offline clients, and a stated best-effort inventory, querying sessions by account ID is easier to operate and may provide tighter read-after-revoke behavior. **Use it until the recovery invariant crosses a store boundary.** Complexity needs evidence.

Whichever option wins, document retention independently for credentials, inventory presentation, risk observations, and security audit records. Deleting an expired credential need not erase the fact that recovery revoked it; retaining a fingerprint indefinitely is not justified merely because the inventory keeps a friendly label. Set limits, test deletion, and make reconciliation observable.

## References

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
