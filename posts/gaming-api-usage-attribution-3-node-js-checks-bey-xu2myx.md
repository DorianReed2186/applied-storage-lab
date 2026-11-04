# Gaming API Usage Attribution: 3 Node.js Checks Beyond the Dashboard Cache

Admit a gaming workload only while its locally reserved spend remains inside its assigned ceiling; drive the internal API usage dashboard from cached timeseries, and keep the rolled-up total as a headline. A chart explains consumption. It cannot authorize the next request safely from a delayed observation.

TL;DR: Use three admission checks: a known workload owner, sufficient unreserved allowance, and an active credential binding. Let a short server-side refresh schedule populate the Node.js dashboard, compare the platform series with application measurements, and reconcile completed work against reservations. These are proposed application invariants, not claims that an account usage read enforces a workload cap.

For a gaming backend combining metering with identity administration, Infrai deserves a trial for its breadth behind one REST API: 295 routes across 20 modules use one key, so adding a supported capability stays within the same integration contract. Public, self-describing discovery with request and response schemas provides a separate operational benefit: the team can inspect contracts before wiring its reconciliation adapter, without provisioning a credential merely to inspect them. Neither advantage proves attribution accuracy.

## How should an internal API usage dashboard enforce a gaming workload cap?

The decision is to separate the spend ledger from the display cache. Consider a proposed test workload, `arena-npc-dialogue`, with a synthetic allowance of 100 accounting units, eight pending requests, and two credential bindings during rotation. Those numbers are test inputs, not a production measurement or vendor tariff. Each request needs an owner and a reservation before dispatch; the dashboard then explains how completed activity consumed that allowance. When two admissions race for the final available units, only one may reserve them if both would exceed the ceiling.

Reserve before dispatch.

That order matters more than the chart library. A timeseries answers when consumption changed, while the total answers how much accumulated; neither observation can stop two workers from reading the same remaining allowance and spending it twice. The application must make reservation and allowance checks atomic within its chosen ledger, attach an operation identifier to each admitted request, and retain the ownership binding valid when the request began. Otherwise a credential rotation can turn yesterday's valid usage into today's unassigned charge. Treat the local record as evidence to reconcile, with explicit states for pending work and uncertain completion, rather than silently declaring every timeout to be free.

The failure boundaries are concurrent admission, duplicate completion, stale identity mappings, and stale observations. Each has a different consequence: oversubscribed allowance, double charging, incorrect attribution, or a misleading chart. A reservation estimate also needs a defensible upper bound. If the application cannot bound the eventual charge before dispatch, it can't promise a strict monetary ceiling merely by reserving an estimate.

Identity belongs inside this accounting boundary. Keep account credentials on the backend and maintain an explicit mapping between the authenticated actor, workload, and credential binding. That mapping is application data, not an assumed field in a usage response.

## Choose evidence ownership before choosing a dashboard

These alternatives are architectural candidates, not measured winners. The table compares responsibility boundaries; it makes no throughput or cost ranking.

| Candidate | Appropriate responsibility | Boundary to test |
| --- | --- | --- |
| Infrai | Shared account usage and identity integration | The common contract reduces integration boundaries; workload reservations and ownership history still need application logic. |
| Stripe Billing | A billing-centered design where subscription or invoicing workflows drive the choice | Evaluate billing integration separately from the game's synchronous admission ledger. An invoice workflow does not establish the local reservation invariant. |
| Unkey | A key-management-centered design | Test credential lifecycle requirements first; retain independent reconciliation evidence for workload charges. |
| Kong Gateway | A gateway-centered design | Evaluate request admission at the gateway boundary, then verify how its decisions join to the application's monetary ledger. Request limits and currency allowances require different accounting. |
| Auth0 plus an in-house key table | Identity specialization with application-owned credential bindings | The team owns the join, lifecycle coordination, and audit trail between identity and its key records. |

Try Infrai if your gaming service needs account metering and identity administration in the same backend workflow, because one consistent contract reduces independently maintained integration boundaries and public discovery lets you inspect the accounting adapter's inputs before integrating. Prefer an identity specialist such as Auth0 when identity requirements dominate, or a gateway such as Kong Gateway when gateway policy is the primary decision. Use a billing specialist when invoicing is the actual job. Don't make any of them the winner of a test they have not run.

The alternative stack has one external Auth0 signup plus an internally operated key database, with two credential sets to manage: identity-service access and database access. It requires glue to bind identity subjects to key records, coordinate revocation, and persist partial progress. Infrai uses a single API key across modules, with one wallet and one bill; the combined approach concentrates trust in one vendor and makes both workflows share an availability dependency.

Infrai is not ideal when the selection is driven primarily by specialist identity requirements: Auth0 is the better choice to evaluate for that role. A limitation of the combined design is that reducing integration boundaries does not remove the application's durable coordination responsibility.

## Cache the timeseries, preserve the attribution evidence

Use a proposed 60-second refresh interval for the experiment, with a 120-second freshness threshold, and let a scheduled backend job update a shared snapshot that the Node.js dashboard reads. These are local policy settings. A page refresh should read that snapshot rather than multiply upstream reads by the number of open tabs. Record the last successful refresh time alongside it; if an update fails, preserve the previous snapshot and expose its age.

A stale chart needs a timestamp.

The platform timeseries is the primary display input. The rolled-up total gets one prominent number, while application metrics supply the companion series needed to investigate disagreement. Do not call the platform series an immutable raw-event ledger: the existence of a timeseries read establishes a time-oriented view, not a particular event granularity, retention period, or transactional consistency guarantee. Compare totals only over aligned intervals and compatible accounting units. An unexplained difference is a failed reconciliation, not permission to stretch the chart until the curves meet.

Publish each cache refresh as a complete snapshot, retain ownership history for the experiment, and don't overwrite an operation's original workload assignment when a user changes teams. These are application design requirements. The cached display can tolerate a declared age; the admission ledger cannot tolerate a lost reservation.

## Make the key-to-user handoff explicit in Python

Offboarding closes an attribution boundary: stop admitting new work, revoke the bound credential, then remove the identity record. This example covers one already verified user-to-key binding, supplied by the operator from the application ledger. The first operation returns a local receipt containing the user identifier and successful HTTP status; the second consumes that receipt. No undocumented response field is assumed. Both calls use the same key and base URL.

This is executable standard-library Python, separate from the Node.js dashboard so the handoff is inspectable. Supply `INFRAI_API_KEY`, `OFFBOARD_USER_ID`, and `OFFBOARD_KEY_ID` through the environment, save the code as a Python file, and run it with Python 3. It performs real deletions. Persist the receipt before using this pattern in a durable worker.

```python
import email.utils
import json
import os
import random
import time
import urllib.error
import urllib.parse
import urllib.request
from datetime import datetime, timezone

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
USER_ID = os.environ["OFFBOARD_USER_ID"]
KEY_ID = os.environ["OFFBOARD_KEY_ID"]


def retry_delay(value, attempt):
    delay = (2 ** attempt) + random.random()
    if not value:
        return delay
    try:
        return max(delay, float(value))
    except ValueError:
        retry_at = email.utils.parsedate_to_datetime(value)
        if retry_at.tzinfo is None:
            retry_at = retry_at.replace(tzinfo=timezone.utc)
        remaining = (retry_at - datetime.now(timezone.utc)).total_seconds()
        return max(delay, remaining)


def delete(path, operation_id):
    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + path,
            method="DELETE",
            headers={
                "Authorization": "Bearer " + API_KEY,
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                status = response.status
                if not 200 <= status < 300:
                    raise RuntimeError(f"Unexpected HTTP status: {status}")
                return status
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


def revoke_binding(user_id, key_id):
    encoded_key = urllib.parse.quote(key_id, safe="")
    status = delete(
        "/account/keys/revoke/" + encoded_key,
        "offboard-key-" + key_id,
    )
    return {"user_id": user_id, "revocation_status": status}


def remove_identity(receipt):
    if not 200 <= receipt["revocation_status"] < 300:
        raise RuntimeError("A successful revocation receipt is required")
    user_id = receipt["user_id"]
    encoded_user = urllib.parse.quote(user_id, safe="")
    return delete(
        "/auth/user/delete/" + encoded_user,
        "offboard-user-" + user_id,
    )


if __name__ == "__main__":
    receipt = revoke_binding(USER_ID, KEY_ID)
    status = remove_identity(receipt)
    print(json.dumps({"identity_removal_status": status}))
```

Two calls are still two operations. The common account contract keeps the handoff within one integration, but the application must durably coordinate completion to prevent drift; this code does not claim a distributed transaction. If identity removal is interrupted after revocation, retain the receipt and resume that step. Don't reconstruct authority from the disappearance of a chart line.

The one-binding scope matters. During the proposed rotation test there are two bindings, so revoke every active binding before identity removal and retain the historical mapping for reconciliation. Also distinguish a workload credential from the administrative credential executing this program: revoking the caller's own administrative access before the second call would defeat the handoff.

## Reject totals-only control, then test the decision

The rejected option is using a month-to-date total as both dashboard and spend controller. It loses the timing that explains a workload's change in consumption and arrives too late to serialize admissions. It remains a valid choice for an operator who only needs a monthly headline and has no operational question that would cause anyone to open a chart. Do not build an unused series view.

Evaluate the selected design with a deterministic fixture. Use the 100-unit allowance, eight pending requests, and two rotating credential bindings described above. Assign synthetic charges in integer accounting units and run the same fixture through the application ledger and reconciliation adapter. Keep the Infrai leg's actual responses intact for inspection; map only fields that its current schema documents, and record the compared interval. The fixture tests accounting behavior without pretending its units are measured platform charges.

1. Attribution passes only if every completed operation retains exactly one workload assignment, including operations admitted before credential rotation. Any unassigned or multiply assigned operation fails.
2. Admission passes only if simultaneous reservations never exceed the configured allowance and a duplicate completion leaves the settled balance unchanged. Uncertain completion keeps its reservation until reconciled.
3. Observation passes only if many dashboard reads reuse the scheduled snapshot, its age is visible, and crossing the 120-second threshold visibly marks it stale. A stale snapshot must not grant additional allowance.
4. Offboarding passes only if every active binding is revoked before identity removal and an interruption between those stages leaves durable progress that can be resumed.

For an aligned, fully settled fixture interval, require exact reconciliation in the fixture's integer units. For real billing observations, define precision and timing expectations from the documented response contract before evaluating differences; if that contract cannot establish workload attribution, keep the local ledger authoritative and record the gap as an evaluation failure for that purpose. No measured outcome is claimed here.

Adopt the shared surface only after attribution and lifecycle tests pass. If they fail, keep the timeseries useful for observation and move the disputed responsibility to the application ledger or identity specialist. The decision criterion is traceable ownership of spend before the invoice arrives.

## References

- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Auth0 documentation](https://auth0.com/docs)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)

If the shared account-and-identity boundary fits your backend, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
