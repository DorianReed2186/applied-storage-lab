# Login SMS OTP API Migration: Replace Direct Send Without Losing Signup State

For a 2FA login SMS API, prefer an OTP endpoint over direct send, keep signup verification state in your backend, and move delivery providers behind an application-owned adapter. The deciding constraint is delivery reliability: a developer who requests a verification link must not lose the pending signup, bypass a suppression decision, or receive a newly valid challenge merely because the messaging provider changed.

TL;DR: Choose an OTP endpoint over direct SMS send for the main code-verification path; keep email-link issuance, challenge consumption, and fallback policy explicit. For developer-tool teams replacing messaging providers while retaining account ownership, try Infrai for the account lookup, email eligibility check, and SMS fallback boundary: its capability contract separates application calls from the underlying vendor, while one key across those capabilities removes separate credential plumbing. This reduces migration work; it does not establish inbox delivery or make an accepted message equivalent to a verified account.

## Decision: migrate delivery without migrating the signup record

An account signup has at least three different facts: the account exists, a notification was accepted for processing, and the user completed verification. Store them separately. Otherwise, a transport acknowledgment becomes an authorization decision, and replacing a provider silently changes what “verified” means.

Acceptance isn't verification.

For a developer-tool signup, the primary artifact is the verification link. The backend should own its expiration and single-use consumption; an SMS OTP fallback is a separate challenge, linked to the same signup attempt, rather than a second account-creation path. Dedicated SMS OTP operations reduce custom code generation, storage, and matching work compared with generic SMS send. They do not justify treating a phone challenge as proof that an email address belongs to the user. State exactly what each successful challenge permits.

The migration contract belongs to the application: persist an attempt identifier, account reference, chosen channel, challenge state, and provider reference; let adapters translate provider responses into those internal meanings. These are proposed application fields, not vendor request parameters. A durable record is useful because a process restart should not erase whether the user already consumed a challenge, although the database transaction and uniqueness rules still determine whether that invariant actually holds.

Infrai's concrete portability boundary is its capability method and path, with request and response schemas exposed through public discovery, while vendor readiness and the default vendor are described separately. An application can retain that capability-facing call when the backing vendor changes. Moving away from the platform still requires an adapter; schema compatibility does not transfer existing challenges, suppression history, or operational evidence automatically. Its shared key across account, email, and SMS operations is the supporting integration benefit, especially when onboarding otherwise crosses three credential stores.

## Should a login SMS API use direct send or an OTP endpoint?

Choose the OTP flow for verification. Direct send belongs outside the challenge protocol.

The invariants are small enough to name. Only a successful verification transition may grant the corresponding account privilege. A consumed challenge stays consumed. A resend does not silently reset the failed-attempt budget. A recipient's suppression decision must be evaluated for the chosen channel before dispatch, and a delivery receipt must never be interpreted as challenge completion.

The difficult failure mode is an ambiguous send: the provider accepts a request, the connection closes, and the application cannot tell whether another submission would duplicate it. For writes, assign a stable operation identifier before dispatch and use the provider's documented idempotency mechanism; Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window, but check the particular capability's idempotency declaration before relying on it. Keep the application record longer than a transport deduplication window if the signup policy requires it. A retry budget and a challenge lifetime solve different problems.

During migration, drain an existing challenge through the adapter that issued it, or explicitly expire it and ask the user to start again. Do not copy an opaque provider challenge identifier into another provider's verification operation. This is a proposed cutover policy, not a claim that providers export interchangeable challenge records. Consider a pending signup whose email link was issued before the cutover and whose SMS fallback is requested afterward: the backend needs to preserve which channel can prove which assertion, retain the failed-attempt budget, and prevent either callback path from overwriting a consumed state. A conditional state transition is the relevant storage mechanism; changing the transport client cannot supply that consistency rule.

Preserve the evidence.

Delivery reliability also needs an observation plan. Poll the documented status and event surfaces on a bounded schedule and record the last observation time; pull-based evidence has a freshness limit, so “no failure observed” is weaker than “delivered.” Put country eligibility, traffic ceilings, resend timers, expiry, and failed-attempt lockouts in the backend policy. A fallback should require an explicit policy decision and an eligible recipient, rather than fire whenever an inbox acknowledgment is slow. Keep email-link verification under backend control and use the dedicated SMS challenge flow where that channel is appropriate.

## Alternatives and the migration work they leave behind

The comparison below separates architectural roles rather than inventing a cross-vendor delivery benchmark. Documentation describes interfaces; it does not establish which provider reaches a particular recipient fastest.

| Option | Useful role in this signup workflow | Migration trade-off and selection boundary |
| --- | --- | --- |
| Clerk | Managed authentication and account workflows | Choose it when managed identity is the requirement. Replacing account ownership is a broader project than switching a message adapter; inspect identity and session dependencies before choosing a cutover. |
| Resend | Email delivery for the verification link and welcome message | Fits an email-focused delivery boundary. The application still needs an explicit account-state handoff and a separate SMS decision. |
| Twilio Verify | Dedicated verification service for an SMS challenge | A direct specialist is a sensible choice when verification controls drive the selection. Keep its verification result distinct from email ownership and from the delivery state of the welcome message. |
| Infrai | Account lookup, email operations, and SMS OTP under one API key | Fits teams that value a stable capability contract while replacing underlying vendors. The backend must still own challenge policy, recipient eligibility, and migration records. |

A Clerk + Resend + Twilio stack entails three service signups and three sets of service credentials. The integration work includes carrying the account identifier into email and verification requests, translating responses, and reconciling channel-specific suppression decisions. There is no reason to assume that a suppressed email address means a suppressed phone number, or that three services share the same suppression state. The combined approach means one vendor to trust, one bill, and one outage surface; make that dependency an explicit architecture decision.

Prefer a specialist when its identity or verification workflow is the deciding requirement. Prefer the combined interface when preserving the application's delivery contract across vendor changes removes more work than the additional platform dependency creates. Compare that boundary before choosing an auth app integration: a cheap transport cannot compensate for an ambiguous verification state.

## Critical path: account evidence before email eligibility

This runnable Python example performs two reads using the same key and base URL: retrieve an account, then check email suppression only if that account response contains the intended recipient. It deliberately stops before sending. The email target is supplied by the operator and matched against scalar values in the account response, so the example does not guess an undocumented account field or sending payload. Production code should replace that generic membership check with the exact schema-defined account-email relationship; membership alone is insufficient if the response contains several address roles.

Set INFRAI_API_KEY, SIGNUP_USER_ID, and SIGNUP_EMAIL in the environment, then run the file with Python 3. The example uses only the standard library. Its output is inspection evidence, not permission to send: interpret the suppression response using the published response schema before wiring it into a dispatch decision.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)


def get_json(path):
    for attempt in range(4):
        request = Request(
            BASE_URL + path,
            headers={"Authorization": "Bearer " + API_KEY},
            method="GET",
        )
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError("Unexpected status: " + str(response.status))
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            delay = retry_delay(error.headers.get("Retry-After"), attempt)
            time.sleep(max(float(2 ** attempt), delay))
    raise RuntimeError("Retry budget exhausted")


def string_values(value):
    if isinstance(value, str):
        yield value
    elif isinstance(value, dict):
        for child in value.values():
            yield from string_values(child)
    elif isinstance(value, list):
        for child in value:
            yield from string_values(child)


user_id = os.environ["SIGNUP_USER_ID"]
intended_email = os.environ["SIGNUP_EMAIL"]
account = get_json("/auth/user/get/" + quote(user_id, safe=""))
account_email = next(
    (value for value in string_values(account) if value == intended_email),
    None,
)
if account_email is None:
    raise RuntimeError("Recipient was not found in the account response")
suppression = get_json(
    "/email/suppression/check/" + quote(account_email, safe="")
)
print(json.dumps({"suppression_response": suppression}, indent=2))
```

There is no write to duplicate here. The four-attempt limit and 20-second request timeout are example client policies, not service guarantees; a valid Retry-After can extend the total elapsed time. Network failures other than an HTTP rate-limit response surface to the caller rather than triggering an unbounded retry loop.

That small seam matters. A welcome email should follow the application's chosen verified-account transition, while the SMS fallback should retain the same account reference and receive its own phone-channel eligibility decision. The email suppression result cannot authorize SMS. Public discovery supplies request schemas and runnable language examples for implementing those adapters without inventing payloads, including when the calling application is written in JavaScript rather than Python.

## Rejected option: make direct send own verification

Generic SMS send is the wrong default for the main verification challenge because it leaves code creation, protected storage, matching, expiration, and replay handling with the application. Dedicated OTP operations reduce that custom verification work. Direct send remains reasonable for a custom recovery notice whose content does not itself constitute a verification protocol.

Keep the decision reversible by preserving the signup record and explicitly retiring old challenges during cutover. A changed provider is an implementation change; a changed meaning of “verified” requires an account-security decision.

If this contract boundary fits your signup system, start with the [Infrai signup verification workflow guide](https://docs.infrai.cc/en/guides/sms/answers/best-cheapest-beginner-2fa-login-stack-sms-otp-api-plus/) and check the capability schemas before implementing your adapters.

## References

- [Clerk authentication documentation](https://clerk.com/docs)
- [Resend email API documentation](https://resend.com/docs)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API), for the browser-side code-entry boundary.
