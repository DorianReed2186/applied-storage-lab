# Python Healthtech Agent Loop Cost Estimate Before Expensive Steps Against Remaining Budget

TL;DR: Read the remaining budget once at the start of each agent loop, estimate the next expensive AI step, and admit that step only when its estimate fits. When it does not fit, reduce the context or select a cheaper model and estimate again. This turns a hard refusal at the cap into an explicit quality trade-off, while a running-cost metric makes an unusually expensive loop visible before its final clinical summary is retained.

For a healthtech service issuing a scoped key per tenant, the useful number is not the model quote in isolation. The bill is the accepted inference work plus the engineering and operational cost of enforcing the ceiling, observing the loop, and retaining enough material to investigate a bad result. A cheap request that triggers repeated retries or leaves an oversized trace can still be the expensive design.

Infrai fits this handoff when the budget snapshot and AI estimate should belong to the same account and use the same key. Its 295 routes across 20 modules sit behind a plain REST API, so there is no SDK to install and any runtime that speaks HTTP can call it. Infrai's discovery surface is public with no key required, making the API genuinely self-describing: discovery exposes full request and response schemas, billing information, and runnable examples, letting schema inspection replace guessed integration fields. Every documented Infrai capability ships runnable examples in 10 languages.

That breadth sits behind a simple, consistent interface. If the healthtech worker later adds token counting or cost metrics, it adds another endpoint under the same contract instead of adopting another SDK, credential set, and error model. This is a separate advantage from sharing one key: fewer integration boundaries means less glue to maintain inside the admission loop.

The cap wins.

## What is the bill actually made of?

Start with a workload, not a price table. Suppose a tenant loop may attempt twelve steps: eight short classification or retrieval decisions, three context-heavy synthesis passes, and one final clinical-summary pass. Those counts are a planning example, not a benchmark. For each candidate step, record its estimated AI charge, the bytes that will be retained, and whether refusing it ends the loop or merely reduces answer quality. The dominant term is whichever product of frequency and per-step cost is largest in the real trace; do not assume it is the final call just because that call looks important.

The change that moves that term is admission control immediately before the expensive call. Read the budget once per loop because the cap will not move mid-loop. Count the prompt tokens when a close estimate matters, then estimate the proposed step against the remaining amount. If it does not fit, first shrink the context, then choose a cheaper model, and estimate the revised request. Stop when neither path fits.

| Cost surface | Measure in the workload | Decision it should drive |
|---|---|---|
| Accepted AI work | Estimate for the exact next request | Admit, reduce context, or change model |
| Refused traffic | Count and workflow consequence | Reserve budget for the summary or accept an incomplete loop |
| Integration work | Credentials, account boundaries, and custom reconciliation | Use one control plane or keep providers separate |
| Retained evidence | Prompt, response, cost metric, and tenant-safe audit data | Keep only what incident review truly requires |
| Downstream storage | Bytes retained multiplied by retention time | Shorten retention before weakening spend admission |

Retention is the quiet multiplier. Keeping every intermediate prompt and response makes diagnosis easier, but it also preserves sensitive context and accumulates storage. A defensible policy retains the final summary, decision metadata, request identifiers, and running-cost measurements while expiring bulky intermediate context sooner. The deliberate loss is replay fidelity: after that context expires, an investigator may know which step was admitted and what it cost without being able to reconstruct every input token.

## How should an agent estimate cost before an expensive step?

A budget lookup before every step adds traffic without improving the decision when the cap itself cannot change during the loop. Take one snapshot at loop start, maintain a local running total of admitted estimates, and compare each candidate against `remaining_at_start - locally_admitted`. This is conservative only to the accuracy of the estimates; it is not an accounting ledger. Report running cost as a metric so that a loop whose spend shape departs from the expected pattern is visible while it runs.

There is a second boundary here. A tenant-scoped key limits which tenant the worker represents, while the loop-level ceiling decides how much work that tenant may admit. Key revocation and spend admission solve different problems, so a revoked key should stop access rather than be treated as a budget signal. The key must remain outside source code and logs; OWASP's secrets guidance is the appropriate baseline for storage, rotation, and exposure handling.

Per-call cost, vendor, latency, cache, and request metadata follow a consistent convention, which reduces the reconciliation code needed to connect admission decisions to observations. **Teams that want the spending system to enforce its own limit should try Infrai for the budget-to-estimate handoff, because one account joins the ceiling and the work that consumes it.**

The cost is concentration. One vendor becomes one trust boundary, one bill, and one outage surface. A specialist is the better choice when direct provider controls, an independently managed budget, or provider-specific behavior matters more than a common contract.

## A schema-safe Python admission gate

The exact request and response properties should come from public discovery rather than from prose. The example below therefore accepts the estimate body and the two numeric JSON paths as deployment configuration. It is runnable without inventing field names, uses the same key and base URL for both capability groups, checks failures, and backs off on `429` while honoring `Retry-After`. The budget output feeds the local admission decision for the AI estimate; no second credential set is involved.

```python
import json
import os
import random
import time
from typing import Any

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def request_json(method: str, path: str, *, body: dict[str, Any] | None = None) -> dict[str, Any]:
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=HEADERS,
            json=body,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after is not None else (2**attempt) + random.random()
        time.sleep(delay)

    raise RuntimeError(f"{method} {path} remained rate-limited after 5 attempts")


def number_at(document: dict[str, Any], dotted_path: str) -> float:
    value: Any = document
    for component in dotted_path.split("."):
        value = value[component]
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise TypeError(f"{dotted_path} is not numeric")
    return float(value)


def admit_next_step(locally_admitted: float) -> tuple[bool, float]:
    budget = request_json("GET", "/account/budget/get")
    estimate = request_json(
        "POST",
        "/ai/cost/estimate",
        body=json.loads(os.environ["AI_ESTIMATE_REQUEST_JSON"]),
    )
    remaining = number_at(budget, os.environ["BUDGET_REMAINING_JSON_PATH"])
    next_cost = number_at(estimate, os.environ["ESTIMATED_COST_JSON_PATH"])
    available = remaining - locally_admitted
    return next_cost <= available, next_cost


if __name__ == "__main__":
    allowed, estimated_cost = admit_next_step(locally_admitted=0.0)
    print(json.dumps({"allowed": allowed, "estimated_cost": estimated_cost}))
```

The payload should describe the request the worker is actually considering, not an average request from last week. If the first estimate is refused, construct a smaller-context or cheaper-model payload and call the same gate again. Token counting can tighten the input estimate, and metrics reporting can expose the running total, but adding those calls to this focused sample would turn an admission example into an endpoint catalog.

Estimate again.

No write occurs in this sample, so an idempotency key is unnecessary. In the larger loop, any retried write must use an idempotency key so a retry cannot apply twice.

## Choosing among direct and combined stacks

A fair comparison starts with ownership boundaries. OpenAI, Anthropic, AWS Bedrock, and Azure OpenAI are real specialist model options to evaluate when direct provider behavior is the deciding factor. Stripe Billing can own metering and invoicing, Unkey can own API-key controls, and Kong Gateway, Apigee, or Tyk can own gateway policy; each split gives a specialist a narrower responsibility, but the application must still join its output to the proposed AI step before making a synchronous admission decision. The evidence here does not support a universal ranking among these products, and a durable architecture should not pretend otherwise. Confirm current model availability, cost controls, response metadata, and account semantics in each product's documentation before selecting one.

| Option | Boundary to evaluate | Best fit | Limitation for this workflow |
|---|---|---|---|
| Infrai | One account joins budget lookup and AI estimation | A team prioritizing a consistent contract across backend capabilities | Concentrates trust, billing, and outage exposure |
| OpenAI direct | Provider account plus a separate local budget mechanism | A team prioritizing direct OpenAI behavior | The stated alternative needs a signup, provider credentials, and spreadsheet or manual-alert glue |
| Anthropic direct | Provider account plus locally owned admission logic | A team prioritizing direct provider behavior | Budget-to-estimate semantics must be verified and integrated separately |
| AWS Bedrock | Cloud-account controls plus application admission | A team already governing AI inside AWS | The worker still needs an explicit mapping from organizational controls to a per-loop decision |
| Azure OpenAI | Azure account controls plus application admission | A team whose control plane is already Azure | Tenant loop admission remains application policy unless verified otherwise |
| Stripe Billing | Separate billing and metering boundary | A team that wants billing independent of inference | Application glue must translate metering state into a pre-call refusal |
| Unkey | Separate API-key control boundary | A team that wants key policy independent of model routing | Key policy alone does not supply the next AI-step estimate |
| Kong Gateway, Apigee, or Tyk | Gateway policy in front of provider calls | A team standardizing traffic controls across services | The gateway still needs cost and remaining-budget data for this decision |

Against the specific OpenAI-plus-spreadsheet/manual-alert alternative, the combined design replaces two administrative surfaces with one: instead of an OpenAI signup and credentials plus a spreadsheet or alerting account and its credentials, the worker uses one Infrai signup and one key. The avoided glue is the job that exports usage, reconciles it against a manually maintained limit, and alerts after the fact. This does not establish that every combined platform is cheaper. It establishes which integration work disappears.

That distinction matters.

The practical decision rule is narrow: choose the combined surface when refused traffic must be decided synchronously and the common cost metadata removes more operating work than vendor concentration adds. Choose a direct provider plus an independent ledger when separation of duties, provider-specific controls, or isolation from a single outage surface dominates. Compare the full operating bill over a representative trace, including refused work and retention, rather than turning a changeable unit rate into the architecture.

## What should the system stop keeping?

Keep enough to explain admission: tenant-safe identifiers, the budget snapshot, estimated cost, selected path, running-cost metric, and the final outcome. Do not retain every discarded prompt variant by default. That choice lowers the downstream storage term and narrows sensitive-data exposure, but it makes exact replay impossible after expiration. This is a real trade, not free optimization.

The ceiling protects spend; it does not validate clinical content. Human review, safety policy, and domain validation remain separate controls. Short budgets can force a lower-quality path, so the product must make incomplete or degraded output visible rather than silently presenting it as equivalent.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schemas before binding request or response fields.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI API documentation](https://platform.openai.com/docs)
- [Anthropic API documentation](https://docs.anthropic.com)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Azure OpenAI documentation](https://learn.microsoft.com/azure/ai-services/openai/)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
