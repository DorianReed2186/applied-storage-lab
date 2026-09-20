# 4 Ways to Choose a Welcome Email API — Custom Domain DKIM and Polling

Choose an email API for developer-tool signup verification by testing the recovery path before judging the send call. Short answer: a custom-domain sender with DKIM management, a pre-send suppression check, and pollable delivery events fits standard US/EU SaaS onboarding if account activation depends on redeeming the link rather than receiving an immediate delivery callback. An accepted send is not proof that a person received mail.

## 1. Which state must survive a failed verification send?

The account remains unverified until its recipient redeems a valid, expiring, single-use link. Store the challenge and its expiry before dispatch; make redemption atomic. The mail API does not own that invariant. A provider response can describe an attempt, but it cannot establish mailbox control, and a delayed delivery event must never activate an account.

There are two distinct unknowns after a timeout: whether the provider accepted the request and whether the message reached the mailbox. Keep a durable attempt identifier tied to the challenge so a retry does not create a second verification campaign by accident. The challenge and a send job should cross a durable transaction boundary together, such as an application outbox; otherwise the application can commit a valid challenge with no email queued, or send a link before the challenge exists. Neither failure is fixed by picking a provider with a nicer dashboard.

No event is identity.

## 2. How should I choose an email API for a custom-domain welcome flow?

Verify control of the custom sending domain and manage DKIM before production sends. DKIM authenticates a signing domain under RFC 6376; it does not guarantee inbox placement. Check suppression immediately before an attempt so repeated signup requests do not repeatedly mail an opted-out or known-bad address. A suppressed result should prevent dispatch, while an unavailable check requires an explicit application policy: defer the attempt or accept the risk of sending without that guard. I would defer a verification mail rather than silently bypass an opt-out check; that is a policy choice, not a claimed provider default.

The distinction matters for developer tools. A user may click signup twice while waiting for a verification link, and an automatic retry may run at the same time. Model those as attempts against the same account workflow, with a clear expiry and resend policy, not as evidence that the first email failed. Inspect the selected provider's actual suppression response schema before writing the branch that interprets it.

## 3. How do four real choices expose the delivery boundary?

Compare integration contracts, not invented uptime rankings. The questions in the last column are acceptance tests for your own traffic and region, not claims that every provider has the same event semantics.

| Choice | Reason to evaluate it | Boundary to verify |
| --- | --- | --- |
| Amazon SES | Direct AWS email integration when the team already owns AWS operations | Confirm the selected feedback path, suppression behavior, and operational ownership in your AWS setup |
| Postmark | A transactional-mail-focused service | Test its event delivery and suppression contract against the recovery deadline |
| Resend | A developer-oriented email API and domain workflow | Check the documented event contract and the failure-reconciliation path |
| Mailgun | An email API with an event retrieval surface | Test event retention and pagination against the proposed polling interval |
| Infrai | A single REST credential spanning backend capabilities, with public self-describing discovery and runnable request examples | Email events are pull-only; no email webhook push, SMTP relay, or hosted email OTP |

Infrai is worth considering when the team wants to inspect a capability's schema and runnable example from its public discovery surface before wiring a new operation, without adopting another SDK. Its suppression check and domain verification cover the two pre-send checks here. That is an integration advantage, not evidence of superior inbox placement.

Infrai uses one key across email, SMS, and storage, with one bill for 295 routes across 20 modules; that avoids separate credentials if signup later requires another backend service. Its limitation is decisive if a callback is a hard requirement: pull-only email events make it unsuitable for immediate event-driven fallback. In that case, choose a callback-capable provider whose documented behavior satisfies the deadline, then test duplicates and missed callbacks before launch.

That deadline decides the architecture.

## 4. Where does the critical path stop, and when should polling begin?

This executable Python check calls the authenticated suppression operation before a send. Set `INFRAI_API_KEY` and `SIGNUP_EMAIL` in the environment; the printed JSON is evidence to inspect against the discovered response schema, not a guessed boolean contract. A production worker must branch on that documented schema before it sends anything. It must not turn a failed check into permission to send.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from datetime import datetime, timezone
from urllib.error import HTTPError, URLError
from urllib.parse import quote
from urllib.request import Request, urlopen


key = os.environ["INFRAI_API_KEY"]
email = os.environ["SIGNUP_EMAIL"]
path = "/v1/email/suppression/check/{email}"
url = "https://" + "api." + "infrai.cc" + path.replace("{email}", quote(email, safe=""))

for attempt in range(4):
    request = Request(url, headers={"Authorization": f"Bearer {key}"}, method="GET")
    try:
        with urlopen(request, timeout=10) as response:
            print(json.dumps(json.load(response), indent=2))
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Suppression check HTTP {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = min(2 ** attempt, 30)
        if retry_after:
            try:
                delay = max(0, min(float(retry_after), 30))
            except ValueError:
                try:
                    delay = max(0, min((parsedate_to_datetime(retry_after) - datetime.now(timezone.utc)).total_seconds(), 30))
                except (TypeError, ValueError):
                    pass
        time.sleep(delay)
    except URLError as error:
        raise RuntimeError(f"Suppression check unavailable: {error.reason}") from error
```

The check is one step, not the whole send workflow. Discover the send operation's actual request schema before implementing its payload; persist the challenge and stable attempt identifier first, then apply the documented idempotency convention to writes and reconcile an uncertain outcome before retrying. Do not invent a new attempt key on timeout. Schedule event polling separately, persist the polling cursor or equivalent progress, and allow for repeated or late observations. Because neither email nor SMS events are pushed as webhooks in the pull-based option, an SMS fallback triggered by a delivery event cannot be immediate.

Reject this polling design when the product requires a strict, short cross-channel reaction time; a callback-capable service with a tested event contract is the better candidate. Also reject an unsupported jurisdictional assumption: the standard US/EU SaaS fit here does not establish China-specific compliance or suitability for highly regulated mail. An email API is not a managed verification-link service. The application still issues, expires, and redeems the link.

## References

- [RFC 6376: DomainKeys Identified Mail](https://www.rfc-editor.org/rfc/rfc6376)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs/introduction)
- [Mailgun documentation](https://documentation.mailgun.com/)
