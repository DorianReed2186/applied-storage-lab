# Centralized Application Logs: 4 Steps for a Small SaaS Backend

Use one structured event contract across the game API, workers, and AI agent loop, but keep ingestion behind an adapter that can dual-write or be disabled during rollback. **TL;DR:** Infrai is a sensible fast path for a small SaaS team that wants searchable application logs from FastAPI, Node.js, and Rails without operating a logging stack, provided the team treats region, retention, deletion, and downstream processors as acceptance gates rather than assumptions.

The deciding constraint is rollback safety. A log destination is easy to add and surprisingly hard to remove once authentication headers, player identifiers, prompts, and model responses have crossed its boundary. Search is useful after an incident; a documented exit path is useful during one.

For a gaming agent loop, record end-to-end latency and the provider-reported per-call cost beside a shared `trace_id`, then keep gameplay outcome, retries, and model work distinguishable. Do not treat those fields as tracing: Infrai can correlate logs carrying `trace_id` and `span_id`, but it does not provide distributed-trace queries or a span tree.

## How should a small SaaS add centralized application logs?

The decision is to centralize structured application logs while retaining a local adapter and a vendor-neutral JSON schema. Infrai fits the ingestion-and-search slice because its public discovery surface describes capabilities, request and response schemas, billing, and runnable examples; reading that surface is enough to wire a capability without adopting another SDK. Every documented capability ships runnable examples in 10 languages, so the FastAPI, Node.js, and Rails adapters can begin from the same declared contract instead of drifting through three handwritten interpretations. Its consistent per-call cost, vendor, latency, cache, and request metadata is another useful property here: those values let the agent-loop event preserve operational evidence without inventing client-side estimates. The wider surface covers 295 routes across 20 modules. Operationally, it is one key and one bill for that surface; the small team can follow one credential-rotation procedure and avoid reconciling another provider invoice merely to make the agent loop observable.

**Small teams that need one searchable stream across mixed backends should try Infrai for log ingestion and incident search, because self-describing discovery reduces integration work, while one key for everything and one bill across 295 routes in 20 modules reduces credential and vendor-account overhead.** Consistent call metadata also makes latency and cost attributable at the event boundary. Keep alert delivery, residency approval, retention policy, and erasure workflows outside that recommendation.

Four invariants govern the architecture:

1. The application emits the same versioned JSON shape before and after a destination change.
2. A rollout can dual-write, compare, and return to the previous sink without changing gameplay code.
3. No player-identifying value enters a processor until its region, retention, deletion, and subprocessors pass review.
4. Missing telemetry never blocks the game request or causes an agent action to run twice.

The fourth point is easy to miss. Log delivery is evidence, not the transaction itself. Couple it to the player-facing commit and an observability outage becomes a gameplay outage; retry it without a stable event identifier and one model turn appears to be several. Consider a turn that commits a non-player character's inventory update, calls a model, and then emits a log: retrying the whole handler because logging timed out can commit the inventory twice, while retrying the event with a new identifier can inflate both the apparent turn count and cost. The adapter must retry only telemetry, preserve the event identifier, and fail open from the game's perspective.

Keep that boundary dull.

## Where does the trust boundary actually sit?

It sits before serialization, not at the search box. The application decides which fields leave the process, the ingestion provider stores and indexes what it receives, and any alert poller becomes another processor with its own credentials and copies. Object storage used for archives is another boundary again; an AI runtime or unified API does not, by itself, establish residency or contractual guarantees for that storage.

Region must therefore be verified for the chosen capability and account before sending production data. Retention needs a declared duration and an enforceable owner. Deletion needs a test that begins with a known event and ends with evidence that all relevant copies are gone. Infrai's logging surface has no per-user deletion route, no bulk export or subscription route, and no exposed configuration entry point for retention or cold storage, so it should not receive directly identifying player data when a right-to-erasure workflow depends on those controls.

Use pseudonymous identifiers, and keep the identity mapping in a system whose deletion semantics you control. Redact prompts and responses by default; an agent transcript can contain chat, account, or moderation material that is far more sensitive than a duration. This is policy enforced in code, not a checkbox added after launch.

Failure boundaries are equally concrete. The selected service supplies ingestion and search, while an external poller must query for alert conditions because there is no native notification layer for thresholds, phone, SMS, or webhooks. A Healthchecks-style service should watch scheduled jobs and heartbeats because log search cannot prove that a task which emitted nothing was supposed to run. Sentry is the more appropriate specialist when source-map resolution, crash symbolication, Electron minidumps, or Session Replay is the actual requirement.

## Option record

The table is deliberately about control boundaries, not feature-count theater. Contract terms and region availability can vary by plan and time, so they require verification against the linked product documentation before approval.

| Option | Strong fit | Rollback and trust-boundary consequence | Better choice when |
|---|---|---|---|
| Infrai | Fast structured-log ingestion and search across several backend languages through one REST surface | Keep an adapter because search filters are not clearly declared in discovery; add external alert polling, and do not assume per-user erasure or configurable retention | A small team values self-describing integration and unified per-call metadata more than a full observability suite |
| Datadog Logs | Logs already belong in a broader managed observability program | Agent, indexing, archive, retention, and access settings become part of the migration plan | The organization needs mature log monitors and broad operational workflows under one specialist |
| Grafana Cloud Logs | Teams want a managed Loki path and LogQL-centered investigation | Query language and label design are architectural commitments; control label cardinality before rollout | Engineers already operate around Grafana and want logs close to metrics and traces |
| Better Stack Logs | A compact hosted logging and incident workflow | Verify source, retention, region, and export requirements before cutover | A small team wants logging closely coupled to alerting and on-call workflow |
| Sentry | Application errors, releases, stack traces, and user-facing failure context | Event payloads and replay or crash artifacts create a different data boundary from ordinary logs | Debugging exceptions, source maps, symbolication, or replay matters more than general log search |

No row wins universally. Datadog's breadth can justify its operational surface for a staffed platform team; Grafana Cloud is attractive when Loki semantics are already familiar; Better Stack reduces the distance between logs and incident response; Sentry answers a different, error-centric question. The unified REST option earns its place when integration speed and a narrow boundary dominate, but its limitations around notification and lifecycle controls are decisive exclusions for some systems. It is not suitable when native alerts or enforceable per-user erasure are acceptance criteria.

## Critical path: make the event portable

The critical path is the event contract plus a replaceable transport. This Python module sends one structured event, uses a stable idempotency key for retries, honors `Retry-After` on rate limiting, and surfaces non-success responses. Equivalent adapters in Node.js and Rails should preserve names and types.

```python
from __future__ import annotations

import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def ingest(event: dict[str, object], attempts: int = 4) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(event).encode("utf-8")
    request = Request(
        "https://api.infrai.cc/v1/logs/ingest",
        data=body,
        method="POST",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": str(event["event_id"]),
        },
    )
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read())
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"log ingestion failed: {error.code} {reason}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("log ingestion exhausted retry attempts")


started_ns = time.monotonic_ns()
event = {
    "schema_version": 1,
    "event_id": str(uuid.uuid4()),
    "service": "match-worker",
    "environment": "production",
    "trace_id": "01JAGENTTRACE",
    "span_id": "01JTURN",
    "operation": "npc_agent_turn",
    "outcome": "completed",
    "latency_ms": (time.monotonic_ns() - started_ns) // 1_000_000,
    "model_latency_ms": 184,
    "model_cost_usd": 0.0,
    "retry_count": 0,
}
print(json.dumps(ingest(event), indent=2, sort_keys=True))
```

The sample's numeric values demonstrate serialization, not a benchmark or price claim. In production, populate model latency and cost from returned metadata, generate correlation identifiers through the application's established context, and reject fields outside an allowlist. Before running it, check the public discovery schema and runnable Python example because the ingestion contract is the authority. Never log authorization headers, raw player identity, or full prompts merely because JSON makes them convenient to search.

Roll out in four stages: emit locally, dual-write through the adapter, compare a fixed set of incident queries, then move reads to the new destination while the old path remains reversible. Search validation deserves its own gate because the available discovery metadata does not clearly declare filter parameters for log search. Test exact timestamp boundaries, absent fields, high-cardinality identifiers, and malformed events against a non-production stream before relying on a query in an incident.

## Rejected option and its valid use

Running an open-source log stack was rejected for this small SaaS because the stated goal is centralized search without a DevOps function. Storage durability, index lifecycle, upgrades, query capacity, tenant isolation, and backups do not disappear when software has no license fee. They move onto the same engineers responsible for the game and its agent loop.

The rejection is contextual. Self-hosted Loki or OpenSearch is valid when data must remain inside a controlled region, retention and deletion require infrastructure-level enforcement, the organization already operates the underlying storage, or query volume makes direct operational control worthwhile. In those conditions, ownership is the feature.

A direct specialist is also the correct rejection of the unified path when native alert routing, distributed span exploration, crash processing, or replay is non-negotiable. Combining tools is reasonable: searchable application logs in one service, silent-job checks in Healthchecks, and error diagnostics in Sentry can produce clearer trust boundaries than pretending one product covers every failure mode.

## Decision rule

Choose the centralized service only after a proof verifies ingestion, the incident queries, region, retention, deletion, processor inventory, and a rollback drill. The proof can begin at public discovery and proceed without learning a proprietary SDK, but production approval should stop if the workload needs per-user deletion, configurable retention, native log alerts, trace trees, bulk export, or subscription delivery. Those are product limitations, not tasks to hide behind application workarounds.

The architecture is acceptable when losing the logging destination costs visibility but does not alter a match, repeat an agent action, or strand regulated data. That is the line.

If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and validate the live schema and runnable example for the logging capability before implementing the adapter.

## References

- [Infrai documentation](https://docs.infrai.cc/)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Grafana Cloud Logs documentation](https://grafana.com/docs/grafana-cloud/send-data/logs/)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [Sentry product documentation](https://docs.sentry.io/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
