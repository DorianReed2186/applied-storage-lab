# Health Check Endpoints Explained: 3 Readiness, Liveness, and Uptime Signals

For a media notification service, an Express health check endpoint must distinguish readiness from liveness without turning every Postgres or Redis wobble into a Node.js restart, and its surrounding uptime signals must retain enough context to attribute operational cost to the tenant or delivery channel that caused it. A single green endpoint cannot do both jobs.

Short answer: expose a cheap liveness endpoint for process survival, a bounded readiness endpoint for traffic admission, and a separate synthetic delivery probe for simple uptime monitoring; keep Postgres, Redis, and channel-level cost signals out of liveness, then correlate all three layers with low-cardinality metrics.

This is a contract, not a dashboard trick. The catch is that a managed uptime monitoring SaaS can check public reachability well, but it is not suitable as the sole judge of queue health, tenant attribution, or whether a notification can actually traverse the delivery path.

## Cost attribution starts before the health check

An Express health check endpoint should answer one narrow question per route. `/livez` asks whether the Node.js process can still serve requests. `/readyz` asks whether this instance should receive new notification work. A synthetic probe asks whether a controlled notification can move through the externally meaningful path. Combining those questions makes the response easy to display and hard to act on.

Keep liveness boring.

A liveness handler should avoid Postgres and Redis calls because a dependency timeout can then cause the orchestrator to restart an otherwise healthy process, increasing load while the dependency is already constrained. Readiness may check the dependencies required to accept new work, but each check needs a strict timeout and an explicit state. For this media workflow, the useful states are ready, degraded, and unavailable: a temporarily unavailable optional analytics sink should not have the same admission consequence as losing the durable store for notification jobs.

The distinction also prevents a common false positive. Suppose HTTP is responsive, Postgres accepts a query, and Redis accepts a ping, yet a channel worker has stopped consuming. Both endpoint checks can be green while delivery is dead. The synthetic probe exists for that gap; it submits a controlled canary, observes completion within a declared window, and never uses a real subscriber address. If the canary is charged or rate-limited like production traffic, tag it as synthetic so cost reports don't leak probe spend into a customer account.

I wouldn't put a tenant ID in the health URL or metric labels. That can expose identifiers, and an unbounded tenant label creates a cardinality problem. Attribute cost in the delivery event stream instead, using stable dimensions such as channel and workload class, then join or aggregate against tenant ownership in the billing pipeline. The health plane should tell operators which capability is impaired; the event plane should tell finance and engineering who consumed what.

## What should a simple Express Node.js health check endpoint prove?

The three signals have different audiences and failure modes. Treating them as interchangeable is where simple uptime monitoring becomes misleading.

| Signal | Question answered | Dependencies | Failure action | Cost-attribution role |
| --- | --- | --- | --- | --- |
| Liveness | Can this process answer? | In-process state only | Restart after repeated failure | None |
| Readiness | Can this instance safely accept work? | Required Postgres and Redis paths, each bounded | Remove from traffic; preserve process for diagnosis | Identify the impaired capability, not the tenant |
| Synthetic delivery | Can a controlled notification complete? | Public ingress, queue, worker, and test destination | Page or open an incident after confirmation | Record a synthetic workload class and channel |

Readiness needs policy, not just connection checks. A Redis failure might block rate limiting and therefore make admission unsafe; in another design Redis may hold disposable response caches, making degraded service acceptable. Postgres deserves the same scrutiny. A successful socket connection proves less than a bounded operation on the exact path required for accepting a notification, but a deep query can itself become load. There isn't a universal dependency list. Document which capability each check protects, its timeout, and why failure closes or leaves open the readiness gate.

Use `200` for the state that satisfies a probe's contract and `503` when readiness is unavailable; keep the response small and stable. Don't return credentials, connection strings, hostnames, stack traces, queue payloads, or tenant identifiers. Those details belong in authenticated telemetry. A compact response can name checks and states without exposing their configuration.

Cost attribution changes the metric design. Export counters for completed and failed notification attempts, partitioned by a bounded channel value and workload class; calculate tenant-level allocation from durable events rather than placing arbitrary tenant identifiers in a time-series label. Prometheus naming guidance recommends a consistent application prefix, base units, and names that describe the measured quantity. Following that pattern, `notification_delivery_attempts_total` reads as a cumulative count, while duration belongs in seconds rather than milliseconds.

## Implement the readiness and liveness contract with an external Python probe

The application handlers remain small; the following external probe is deliberately Python because it tests the HTTP contract without sharing the Express process or its dependencies. Run it from a different failure domain than the service. It checks liveness and readiness independently, applies a two-second timeout, and emits one line per result that a scheduler or log collector can consume.

```python
from __future__ import annotations

import json
import sys
import time
from dataclasses import asdict, dataclass
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class ProbeResult:
    probe: str
    ok: bool
    status: int | None
    duration_seconds: float
    error_class: str | None


def probe(base_url: str, name: str, timeout_seconds: float = 2.0) -> ProbeResult:
    started = time.monotonic()
    request = Request(
        f"{base_url.rstrip('/')}/{name}",
        headers={"User-Agent": "notification-health-probe/1"},
    )
    try:
        with urlopen(request, timeout=timeout_seconds) as response:
            status = response.status
            response.read(4096)
            return ProbeResult(
                probe=name,
                ok=200 <= status < 300,
                status=status,
                duration_seconds=time.monotonic() - started,
                error_class=None,
            )
    except HTTPError as exc:
        return ProbeResult(
            probe=name,
            ok=False,
            status=exc.code,
            duration_seconds=time.monotonic() - started,
            error_class=type(exc).__name__,
        )
    except (TimeoutError, URLError) as exc:
        return ProbeResult(
            probe=name,
            ok=False,
            status=None,
            duration_seconds=time.monotonic() - started,
            error_class=type(exc).__name__,
        )


def main() -> int:
    base_url = sys.argv[1]
    results = [probe(base_url, name) for name in ("livez", "readyz")]
    for result in results:
        print(json.dumps(asdict(result), separators=(",", ":")))
    return 0 if all(result.ok for result in results) else 1


if __name__ == "__main__":
    raise SystemExit(main())
```

The exit code makes the probe usable in a scheduled job, but don't collapse the two records when alerting. `livez=failed` suggests a process problem; `readyz=failed` with liveness intact suggests traffic should drain while the process remains available for diagnosis. The error class is intentionally coarse. Detailed dependency errors should be captured by the service's protected logs and traces, where access control and retention policy can be applied.

This probe does not perform the synthetic notification. That check needs its own identity, destination, timeout, cleanup rules, and budget because it creates real work. I'm not sure one delivery window fits every channel; measure normal completion distributions for each channel, then set a window that catches material delay without paging on routine variance.

## How should a team choose its monitoring boundary after defining failure policy?

Choose the operating model after defining the signals. Each option has a legitimate boundary, and mixing them is often more defensible than pretending one tool sees the whole path.

| Model | Useful when | Limitation | Cost implication |
| --- | --- | --- | --- |
| Orchestrator probes | Restart and traffic-removal decisions must stay close to the workload | They usually observe one instance, not end-to-end delivery | Low direct probe work; no tenant allocation |
| Self-hosted metrics and probes | The team needs control over labels, retention, and private-network checks | The team owns storage growth, upgrades, and alert delivery | Infrastructure cost must be allocated explicitly |
| Managed uptime monitoring SaaS | Public reachability and independent external checks matter | Private dependencies and tenant event context may be outside its view | Subscription and probe usage form a separate cost center |
| Synthetic delivery worker | Queue, worker, and destination behavior must be tested together | It creates traffic and requires safe test recipients | Mark every attempt as synthetic and allocate it to operations |

A managed SaaS is a poor fit when the service is private, policy forbids an external probe, or cost attribution requires joining delivery events with internal tenant ownership. Stick with an internal probe and self-hosted telemetry in those cases. Conversely, an internal-only monitor shares too much fate with the system it watches; an independent external check can detect edge, DNS, and reachability failures that an in-cluster check cannot see.

No single green light is enough.

Alert rules should preserve the distinction: page on sustained synthetic delivery failure or broad readiness loss, route isolated dependency degradation to the owning team, and treat one failed probe as evidence to confirm rather than a complete incident diagnosis. Exact thresholds depend on traffic, channel latency, retry policy, and error budget. Your mileage may vary, especially for low-volume channels where a ratio can swing on one event.

## Migrate to three signals in controlled stages

Start by defining `/livez` as an in-process check and `/readyz` as a bounded admission decision. Deploy them without automated restarts first, observe their state transitions, and verify that a dependency slowdown cannot make the check consume unbounded connections. Then connect readiness to traffic removal. Connect liveness to restart behavior last, with repeated-failure protection, because a bad liveness dependency can create a restart loop.

Next, add the external probe and a synthetic notification for one test destination per materially different channel. Tag those delivery events with a fixed synthetic workload class, verify they are excluded from tenant invoices, and include their actual resource use in the operations cost center. Test four cases before relying on the alerts: process unavailable, required database path unavailable, Redis-dependent admission unavailable, and a worker that accepts work but does not complete it. The expected outcomes should differ across liveness, readiness, and synthetic delivery; if every case flips every signal red, the contracts are still coupled.

Finally, review labels before traffic grows. Channel and workload class are bounded enough to support operational breakdowns; raw tenant ID, destination, message ID, and error text are not. Keep those high-cardinality fields in durable delivery records and protected logs. This split gives uptime monitoring a stable signal while preserving the event detail required to explain cost.

Ship the contract in stages.

## References

- https://prometheus.io/docs/practices/naming/
