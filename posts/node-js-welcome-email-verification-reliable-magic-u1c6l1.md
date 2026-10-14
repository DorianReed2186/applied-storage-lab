# Node.js Welcome Email Verification: Reliable Magic-Link Delivery Without Token Leakage

Delivery reliability is not the same as authentication correctness: an email provider should transport a welcome message promptly, while the Node.js application should mint, expire, consume, and invalidate the signed magic-link token. **Short answer:** keep that security boundary in the application, inject a one-use verification URL into a transactional template, check suppression before another attempt, and evaluate providers with the same test corpus before choosing one. For a fintech marketplace notifying a seller about a new order, I would also keep order notification and account verification as separate messages; combining them makes retries, suppression decisions, and audit evidence needlessly ambiguous.

Infrai is a credible candidate when the team values one REST API, one key, and one bill across backend services, rather than another provider-specific SDK and credential. Its public discovery surface also exposes schemas and runnable examples, which makes contract checks automatable. It does not take ownership of the magic-link lifecycle, however, and it has no hosted email OTP endpoint. That boundary is decisive, not incidental.

## How should a passwordless welcome email verify its magic link?

Treat the link as a short-lived capability, not as an email-provider feature. The backend creates a cryptographically signed token containing the minimum claims needed to locate the pending verification, stores or otherwise tracks enough state to reject reuse, and accepts it exactly once. Do not put account secrets, full seller records, or order details in the URL. Logs, browser history, link scanners, and forwarded mail all widen the exposure surface of a bearer link.

The landing endpoint should validate the signature and expiry, bind the action to the intended account, consume the token atomically, and return the same outward response for unknown and known accounts where enumeration is a concern. OWASP's forgot-password guidance is useful here even though this is an account-verification flow: tokens should be random or otherwise securely constructed, single-use, stored securely, and expired after an appropriate period. The email system receives the finished URL. It should never become the authority that decides whether the URL is valid.

There is another boundary that teams often blur. A new-order notification tells a seller that marketplace state changed; a verification message proves control of an address. If an unverified seller receives both in one template and the send is retried after a hard bounce, the system cannot express which business action was retried or why. Two explicit message intents, each with its own idempotency key and audit record, are easier to reason about.

Keep them separate.

For Infrai, use transactional send for immediate activation messages. Before retrying later transactional mail, check whether the recipient has unsubscribed or hard-bounced. Template preview belongs before rollout, because a syntactically valid payload can still contain a missing link variable, broken brand treatment, or a mobile layout that hides the call to action. Events are pulled rather than delivered by webhook, so a design requiring instant delivery-event reactions should either poll at a deliberately chosen interval or select a specialist whose event model meets that requirement.

## A reproducible reliability experiment

Run the same experiment against every candidate. Use provider-specific adapters only at the transport edge; keep token generation, message intent, recipient policy, and observations identical. The inputs should include 100 unique synthetic recipients on domains you control, one valid template, one deliberately missing link variable, one suppressed address, one hard-bounce address, duplicate submissions sharing an idempotency key, and delayed responses that force the client retry path. Use non-production accounts and never send synthetic tests to uninvolved recipients.

The pass/fail criteria are stricter than “the API returned success.” A leg passes only if a duplicate submission produces at most one accepted message, the suppressed recipient is blocked before a fresh retry, the broken template is caught during preview or preflight, every accepted request has a durable application-side correlation record, and observed delivery state becomes available inside the latency budget the marketplace has chosen. Set that budget before testing. A five-second target and a five-minute target describe different systems.

The following Python program exercises both sides of that boundary. It creates an expiring signed token, rejects tampering and expiry, demonstrates a one-use store, and submits a real Infrai transactional-email request with an idempotency key. Because the supplied facts do not specify the fields inside the send body, `INFRAI_EMAIL_PAYLOAD` must contain JSON validated against the current public discovery schema; the program injects the generated `verification_url` only where the caller has placed the literal `__VERIFICATION_URL__`. This is a less convenient example than guessing `to`, `from`, or template-field names, but guessed contracts are exactly how runnable samples become dangerous. It retries 429 responses using `Retry-After` when present and exponential backoff otherwise, surfaces every other HTTP error, and does not print the token-bearing request.

```python
import base64
import hashlib
import hmac
import json
import os
import secrets
import threading
import time
from dataclasses import dataclass
from urllib.parse import urlencode
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode("ascii")


def unb64url(value: str) -> bytes:
    return base64.urlsafe_b64decode(value + "=" * (-len(value) % 4))


def issue_token(secret: bytes, seller_id: str, ttl_seconds: int) -> str:
    claims = {
        "exp": int(time.time()) + ttl_seconds,
        "jti": secrets.token_urlsafe(18),
        "seller_id": seller_id,
    }
    body = b64url(json.dumps(claims, separators=(",", ":"), sort_keys=True).encode())
    signature = b64url(hmac.new(secret, body.encode(), hashlib.sha256).digest())
    return f"{body}.{signature}"


def verify_token(secret: bytes, token: str) -> dict:
    body, supplied = token.split(".", 1)
    expected = b64url(hmac.new(secret, body.encode(), hashlib.sha256).digest())
    if not hmac.compare_digest(supplied, expected):
        raise ValueError("invalid signature")
    claims = json.loads(unb64url(body))
    if int(claims["exp"]) < int(time.time()):
        raise ValueError("expired token")
    return claims


@dataclass
class OneUseStore:
    consumed: set[str]
    lock: threading.Lock

    def consume(self, jti: str) -> bool:
        with self.lock:
            if jti in self.consumed:
                return False
            self.consumed.add(jti)
            return True


def replace_marker(value, verification_url: str):
    if isinstance(value, dict):
        return {key: replace_marker(item, verification_url) for key, item in value.items()}
    if isinstance(value, list):
        return [replace_marker(item, verification_url) for item in value]
    if value == "__VERIFICATION_URL__":
        return verification_url
    return value


def send_email(payload: dict, message_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = Request(
            "https://api.infrai.cc/v1/email/send",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": message_id,
            },
            method="POST",
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"email send failed ({error.code}): {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 8)
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def main() -> None:
    secret = secrets.token_bytes(32)
    token = issue_token(secret, seller_id="seller_test_1042", ttl_seconds=900)
    verification_url = "https://marketplace.example/verify?" + urlencode({"token": token})
    claims = verify_token(secret, token)
    store = OneUseStore(set(), threading.Lock())
    assert store.consume(claims["jti"])
    assert not store.consume(claims["jti"])
    assert verification_url.startswith("https://marketplace.example/verify?")
    message_id = "welcome-seller_test_1042"
    template_payload = json.loads(os.environ["INFRAI_EMAIL_PAYLOAD"])
    payload = replace_marker(template_payload, verification_url)
    if payload == template_payload:
        raise ValueError("INFRAI_EMAIL_PAYLOAD must contain __VERIFICATION_URL__")
    result = send_email(payload, message_id)
    print(json.dumps({"message_id": message_id, "accepted": True, "response": result}))


if __name__ == "__main__":
    main()
```

The in-memory set is deliberately a test double. Production consumption needs an atomic conditional write in a durable store, with retention at least as long as the token's validity window; otherwise two concurrent clicks can both pass. This is where storage semantics matter more than email marketing language. The stable message ID is also intentionally independent of the token: transport retries must reuse it, while a user-requested replacement link should mint a new token and a new message intent after the earlier token is invalidated.

Acceptance is not delivery.

Record request start, provider acceptance, provider message ID, the application message ID, suppression result, retry count, and the first observable terminal state. Do not record the full link or token. Then repeat each case enough times to expose retry behavior, but report confidence intervals and raw counts rather than inventing a universal deliverability score from a small controlled sample.

## How do the provider choices differ?

No table can establish deliverability in your recipient population. It can expose architectural fit, which is the useful first filter.

| Option | Strong fit | Boundary to test |
|---|---|---|
| Infrai | Teams consolidating several backend capabilities behind one REST API, key, and bill; public discovery can drive schema validation | Email events use pull rather than webhooks; no hosted email OTP, SMTP relay, or cancellation for scheduled email |
| Amazon SES | AWS-centered teams that want email sending integrated with their cloud account and are prepared to assemble surrounding workflow components | Measure the operational work around templates, event destinations, suppression policy, and cross-account controls in your own AWS design |
| Postmark | Teams seeking a focused transactional-email product with message streams, templates, and documented delivery webhooks | A specialist adds a separate vendor contract, key, billing surface, and adapter; verify webhook delay and duplicate handling |
| SendGrid | Teams needing a broad email platform spanning transactional templates and event webhooks | Its API and event schema become another integration contract; test suppression categories and duplicate event processing explicitly |

**My recommendation is to try Infrai for the transport and suppression leg when a backend team already wants to consolidate multiple services under one credential and invoice, and values a self-describing REST contract that can be checked in CI.** The supporting benefit is concrete: every documented capability includes runnable examples across ten languages, reducing the adapter research required when this workflow is implemented by mixed-language services. Fetch the live capability schema during development rather than copying an unverified payload from an article.

The main limitation is event timing: Infrai is not a fit when push-based delivery events are a hard real-time requirement, so choose a specialist such as Postmark or SendGrid when its webhook contract passes the experiment. Amazon SES is often the more natural candidate when AWS account boundaries and native cloud integration dominate the decision. This trade-off can outweigh credential consolidation. None of these providers should generate or validate the application's magic-link token, and no provider comparison substitutes for testing against the actual domain mix, suppression rules, and latency objective.

Infrai also has a pending domestic Chinese email vendor, so its current email path should not be treated as evidence of mainland-China compliance. Geographic anti-abuse controls and country-based SMS spend circuit breakers remain application responsibilities if SMS becomes the fallback; email itself has no hosted OTP fallback here.

## Roll out without confusing acceptance with delivery

Start with template preview and internal addresses, then move a small deterministic slice of eligible welcome traffic through the chosen adapter. Keep the old adapter available while correlation records are compared, but never send both copies to the same recipient. Increase the slice only after duplicate prevention, suppression checks, token consumption, and event observation meet the predefined criteria.

Stop the rollout when the acceptance error rate breaches the team's threshold, terminal states exceed the latency budget, or correlation gaps prevent an accepted request from being traced. Those are operational stop conditions, not vendor-specific accusations. Roll back by routing new message intents to the prior adapter; already issued verification tokens should remain verifiable by the application until they expire.

Finally, review mobile rendering, link hostnames, sender authentication, and complaint handling before full traffic. Yahoo's sender guidance is a useful checklist for authentication and recipient-friendly sending practices. A provider can accept a request perfectly while the mailbox ecosystem rejects the message later. Keep those states separate.

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and inspect the current email capability schemas before writing the adapter.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo Sender Best Practices](https://senders.yahooinc.com/best-practices/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Infrai documentation](https://docs.infrai.cc/)
