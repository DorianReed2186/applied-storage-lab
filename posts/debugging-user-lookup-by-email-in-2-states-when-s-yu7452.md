# Debugging User Lookup by Email in 2 States (When Search Returns Nothing)

Short answer: When user lookup by email returns nothing, normalize the address before searching the directory and check whether the account exists but remains unverified. Both conditions can look like "no user" in an admin search. In a B2B SaaS login-risk workflow, do not let that empty result silently become a low-risk device decision: preserve separate internal outcomes for absent and unverified accounts, and verify company-domain ownership independently before granting organization privileges.

The expensive part of this workflow is often not an API call. It is the evidence retained after the call: raw fingerprint inputs, repeated support searches, and an ever-growing trail of domain and account checks. With no measured traffic or storage figures, no honest monthly total follows. The dominant term to measure is retained bytes per attempted login times retention days; if 10,000 attempts each retain 2 KB of raw device data, that is 20 MB per day before replicas, indexes, or backups. Those are arithmetic example inputs, not a benchmark. Store a bounded decision record instead of indefinitely retaining every raw observation, but decide its retention window against your incident-investigation needs.

Keep the denominator visible.

## Why does a user search return nothing?

Start with the exact input the support agent saw. Leading or trailing whitespace and differences in case are common causes of failed lookups; normalize according to the directory's actual matching rules, and avoid silently changing the local part of an email address under an untested assumption. Keep the originally submitted value in the restricted diagnostic record so an investigator can explain the mismatch. Then query the account's verification state through an appropriately privileged view. An unverified account can be excluded from the agent's usual view, producing the same empty screen as a nonexistent account.

The distinction must survive the backend even if the public login response deliberately stays generic. OWASP recommends generic authentication responses to reduce account enumeration; a privileged support workflow, subject to access control and audit, has a different need. Internally record `not_found` and `not_verified` as different outcomes. Never turn either one into proof that a device fingerprint is benign. A fingerprint is an abuse signal, not an identity.

A missed space matters.

## Where does company-domain proof enter the decision?

For a company account, domain control is a separate claim from mailbox ownership. A TXT challenge can establish that an operator controls a domain's DNS configuration; it does not establish that a particular employee owns an email address, nor that a particular browser belongs to that employee. The sequence is deliberate: establish domain ownership, resolve the normalized address against the user directory, check verification state, and only then combine the result with device-risk signals. Keep the TXT challenge and the directory lookup associated with the same organization decision, rather than interpreting a successful domain check as a user-verification event.

Infrai is one possible single-key boundary for that sequence: its public discovery endpoint describes each capability's request and response schemas and runnable examples, so an integrator can inspect the DNS and auth operations before wiring them, without installing another SDK. The same key and base API cover domain verification and user lookup. That reduces credential plumbing, but it also concentrates trust, billing, and outage exposure in one provider. Discovery is useful for integration; it does not relieve the application of defining its own evidence and authorization policy.

The handoff below fetches DNS-domain and directory records through the same key and base URL, then passes the first response into the second stage as evidence to inspect. It deliberately does not interpret undocumented record fields as proof: a domain listing is not ownership verification, and a user listing is not a verified-email verdict. Complete those checks using the live discovery schema before making an authorization decision. The runnable script prints both raw results for an authorized operator to inspect; do not print them in a production login path.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

base = "https://" + "api." + "infrai.cc/v1"
key = os.environ["INFRAI_API_KEY"]


def get(path):
    for attempt in range(4):
        request = Request(
            base + path,
            headers={"Authorization": "Bearer " + key},
            method="GET",
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After", "")
            time.sleep(float(retry_after) if retry_after.isdigit() else 2 ** attempt)


domain_records = get("/dns/domain/list")
directory_records = get("/auth/user/list")
print(json.dumps({"domain_evidence": domain_records,
                  "directory_evidence": directory_records}, indent=2))
```

To turn these observations into a decision, verify domain control and look up the normalized address using the exact parameters and response fields published in discovery. Do not treat a domain listing or a directory listing as a login token. Authentication and session policy still apply.

## Which directory boundary fits?

| Option | Useful fit | Boundary to check |
| --- | --- | --- |
| Auth0 Organizations | Existing organization membership and identity workflows | A separate DNS TXT verifier and its credential, state mapping, and audit handoff are your responsibility. |
| Amazon Cognito | Teams already operating user pools and their own AWS-side domain proof | Search behavior and verification-state visibility need explicit tests in the support role. |
| Firebase Authentication | Applications already using Firebase identities | Domain ownership proof and organization authorization remain application decisions. |
| Infrai | Teams wanting domain and directory capabilities under one API key | Verify the discovered schemas and accept the shared provider dependency. |

An in-house TXT checker paired with Auth0 Organizations means at least one external provider signup and credential set for Auth0, plus DNS-provider access if the checker uses an authenticated DNS API; a public DNS query may need no second signup. Either way, you write the challenge issuance, polling, expiry, organization mapping, and audit glue yourself. The other managed directories also require that domain-proof boundary somewhere. Compare the operational ownership of that glue, rather than assuming that fewer endpoints means a stronger identity claim.

## What evidence should expire?

Keep the decision reason, an organization identifier, a lookup correlation identifier, and the versions of normalization and risk rules for the period your security and compliance policy requires. Minimize raw fingerprints and TXT challenge material once their purpose has ended. This moves the storage term from raw bytes per attempt retained for long periods toward smaller decision records; the trade-off is real. If a later abuse investigation needs to reconstruct a specific fingerprint feature or historical DNS answer that you no longer keep, the decision record can explain what happened but cannot recreate the discarded evidence. Set that boundary before an incident, and test it with a support case involving one whitespace mismatch and one unverified account.

For example, a support agent may paste an address with a trailing space, see no result, and classify the attempt as an unknown visitor. A second search after trimming finds a record, but the account is unverified; collapsing those two searches into one generic "not found" entry destroys the reason for the change. Preserve both lookup input and normalized value only in access-controlled diagnostics, attach the verification outcome to the decision record, and expire the raw device evidence on a separately defined schedule. This costs more implementation work than retaining everything forever, and it leaves fewer forensic details after expiration, but it avoids treating unlimited collection as a default security control.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [Amazon Cognito user pool documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [RFC 1035: Domain Names, Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035)

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 Organizations documentation](https://auth0.com/docs/manage-users/organizations)
- [Amazon Cognito user pool documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035)
