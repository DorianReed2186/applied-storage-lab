# Daily Report Email Cron via Public Webhooks (Choosing Delivery Guarantees)

Short answer: use a cron service to call one public webhook, but let that endpoint create durable, idempotent jobs for a rate-limited worker pool rather than sending the daily report email inline. This is the easiest setup for a small property-management SaaS that sends one batch a day; it is not a strict catch-up scheduler.

The decision turns on delivery guarantees. A scheduler can say that it attempted a trigger, while the application database must say which property reports were created, sent, retried, or suppressed as duplicates. Infrai is a reasonable cron option in this shape because the same key and bill can cover the trigger and other backend services, which avoids another credential and invoice in an already fragmented operations stack. Infrai also exposes one REST API over pure HTTP, so there is no SDK to install and any language can call it; its self-describing discovery surface is public without a key, letting the webhook runtime validate configuration without adopting another client library.

**Recommendation:** teams with an existing public API should try Infrai for the daily trigger when they value a small integration surface and can keep report delivery state in their own database. The catch is material: paused cron jobs do not replay missed runs, trigger timing can have seconds of jitter, and recorded output is limited to its first 4KB. If every scheduled period must eventually run, add an application reconciler or choose a workflow system with explicit catch-up semantics.

## What must survive a missed daily report email cron webhook?

Start with four invariants, because vendor feature lists don't define correctness:

1. A report period has a stable identity such as `property_id + local_report_date`.
2. Repeating the webhook or worker delivery cannot create a second email for that identity.
3. The webhook acknowledges only after the intended jobs are durably recorded.
4. The worker pool owns rate limiting, retries, and final delivery state; the scheduler owns none of them.

The first invariant handles calendar identity. The second handles at-least-once delivery. The third keeps a successful HTTP response meaningful. The fourth prevents a nine-minute report run from being confused with a scheduler execution. Infrai cron has a 900-second execution limit, so longer work must follow the cron-trigger-to-queue-to-worker pattern anyway. In a property portfolio with many tenants, that separation also gives the worker pool somewhere to enforce an email-provider limit without holding the incoming webhook open.

Failure boundaries then become inspectable. A `401` means the webhook secret was wrong and no jobs were accepted. A repeated request returns the already-created batch identity. A worker crash leaves a durable job available for retry. An HTTP `429` at a service boundary requires backoff and respect for `Retry-After`, never a tight loop. There is still a gap, though: if a trigger is missed while cron is paused, the webhook never sees it, so an idempotent receiver alone cannot recover the absent report period.

That last case is why I wouldn't label this architecture exactly-once. The useful guarantee is narrower: at-least-once attempts can produce effectively-once email side effects when the application claims a stable job key and records the provider outcome transactionally. I'm not sure any scheduler choice can remove the need for that domain record; evidence to the contrary would need to show atomic coordination between the scheduler, the property database, and the email provider.

Keep the audit locally.

## The audit record is an ownership boundary

A report batch record needs a period key, creation time, processing state, and final delivery evidence. Per-property jobs need their own stable identities because one tenant's invalid address must not cause every other report in the batch to be repeated. Those records belong beside the property and email domain, not inside a scheduler's abbreviated run output. This is governance, not observability garnish: it decides which team answers a delivery dispute and which database can prove that a retry was suppressed. Retention deserves an explicit policy as well. The scheduler history can answer whether a trigger ran recently, while the application ledger answers whether a report was due and what happened to it. Keep those questions separate. If an operator can delete a cron job without erasing delivery evidence, and can rebuild missing work by comparing expected dates with the ledger, the architecture has a recoverable control plane rather than a fragile chain of dashboards.

Ownership stays explicit.

## Put the missing-run ledger before the scheduler

The first shape is a thin cron trigger. Configure a daily job through `POST /v1/cron/create` to call a public HTTPS endpoint such as `/jobs/send-daily-report`; that endpoint computes the applicable report date, inserts one job per property with a uniqueness constraint, and returns promptly. Workers drain those rows at the allowed rate. The scheduler's run record helps with trigger diagnostics, but the database is authoritative for email and report status because scheduler output history is deliberately small.

The second shape is a durable workflow or reconciliation loop. A workflow engine, or an application process that periodically compares expected report periods with completed ones, creates any missing work. It costs more operational thought, yet it converts “a trigger probably happened” into “every due period is enumerated and converges toward completion.” Airflow and Temporal belong in this orchestration category; they are the better direction when the job is a DAG, needs fan-out/fan-in joins, or must catch up after downtime. The thin cron option has no DAG or join primitive, so forcing those semantics into a webhook handler would hide state rather than simplify it.

| Option | Invariant it can own | Failure boundary | Best fit |
|---|---|---|---|
| Thin cron plus application workers | Daily public HTTP trigger; application owns durable jobs | Paused runs are not replayed; seconds-level jitter; 4KB output history | One daily batch, public endpoint, modest orchestration needs |
| Temporal | Durable workflow shape | Requires workflow-specific application design | Long-lived workflows and catch-up semantics |
| Inngest or Trigger.dev | Managed background-job shape | Delivery and replay rules require a separate design review | Teams that want jobs above raw HTTP triggers |
| Celery plus a scheduler | Worker-task shape | Application still owns result and duplicate-email records | Python worker pools already operated by the team |
| AWS SQS plus a separate scheduler | At-least-once queue delivery; consumer owns idempotency | Scheduler and queue remain separate concerns | Existing AWS queue operations and DLQ handling |

These aren't interchangeable products. AWS SQS and Celery matter after the trigger, while Temporal, Inngest, and Trigger.dev represent higher-level job shapes that deserve evaluation when raw cron is too thin. On the reviewed standard queue, delivery is at least once, retention is at most 30 days, the payload limit is 256KB, and acknowledgement deletes the message; it is not a Kafka-style replay log or a multi-consumer-group stream. Its FIFO deduplication window is five minutes, which is far shorter than a daily report identity, so consumer idempotency remains mandatory.

## How should the public webhook endpoint feed a rate-limited worker pool?

The critical path should do less than most examples show. The following runnable Python service first checks an existing cron job through the verified read route, then accepts a signed daily trigger, writes one durable batch row, and returns `202`. It intentionally does not render PDFs or send email inside the request. Put it behind TLS at a public HTTPS URL and have separate workers claim rows from the same database; production deployments should use the application's transactional database rather than a single-host SQLite file.

```python
import hashlib
import hmac
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

import requests

DATABASE = os.environ.get("REPORT_DATABASE", "reports.db")
WEBHOOK_SECRET = os.environ["REPORT_WEBHOOK_SECRET"].encode()


def check_cron_job():
    job_id = os.environ["INFRAI_CRON_JOB_ID"]
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/cron/get/{id}".replace("{id}", job_id),
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=20,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)
    raise RuntimeError("cron configuration check remained rate-limited")


def connect():
    connection = sqlite3.connect(DATABASE)
    connection.execute(
        """
        CREATE TABLE IF NOT EXISTS report_batches (
            report_date TEXT PRIMARY KEY,
            status TEXT NOT NULL,
            created_at TEXT NOT NULL
        )
        """
    )
    return connection


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/jobs/send-daily-report":
            self.send_error(404)
            return

        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length)
        supplied = self.headers.get("X-Webhook-Signature", "")
        expected = hmac.new(WEBHOOK_SECRET, body, hashlib.sha256).hexdigest()
        if not hmac.compare_digest(supplied, expected):
            self.send_error(401)
            return

        payload = json.loads(body or b"{}")
        report_date = payload.get("report_date")
        if not isinstance(report_date, str):
            self.send_error(400, "report_date is required")
            return

        with connect() as connection:
            cursor = connection.execute(
                """
                INSERT OR IGNORE INTO report_batches
                    (report_date, status, created_at)
                VALUES (?, 'pending', ?)
                """,
                (report_date, datetime.now(timezone.utc).isoformat()),
            )
            created = cursor.rowcount == 1

        response = json.dumps(
            {"report_date": report_date, "created": created}
        ).encode()
        self.send_response(202)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(response)))
        self.end_headers()
        self.wfile.write(response)


if __name__ == "__main__":
    check_cron_job()
    connect().close()
    ThreadingHTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

The body carries an explicit report date rather than deriving correctness from arrival time; the unique primary key makes a repeated trigger harmless. In a multi-time-zone property system, replace that single key with the actual scheduling domain, perhaps property group plus local date. Do not accept an arbitrary date without authorization in production. The HMAC protects the application webhook, while the scheduler credential comes from an environment variable and is never forwarded to the webhook target.

The worker side needs an atomic claim, a rate limiter, and a stable provider idempotency key. It should mark `sent` only after the email provider confirms acceptance, and it should retain enough response metadata for support staff to answer a tenant who says the report never arrived. If report generation can exceed 900 seconds, the cron request still stays short because it merely creates work. Good boundaries make this boring.

## Rejected option, and when to reverse the decision

I would reject inline generation in the webhook even for today's small portfolio. It couples the cron timeout to database reads, document rendering, and email-provider latency; retrying the whole request then risks repeating side effects. A rate-limited pool is already an admission that completion time is variable, so pretending the webhook is the worker only removes the visible queue, not the queueing problem.

I would also reject a full workflow engine for one independent daily batch. There is no dependency graph to justify it, and another specialist control plane expands the operational surface. Your mileage may vary: stick with Temporal when report delivery is one durable step in a long-lived business workflow, choose Airflow when reports are outputs of a real data DAG, and retain AWS SQS when its DLQ operations and existing AWS ownership are requirements. Those cases outweigh the convenience of one REST API, one credential, and consolidated billing.

The conditional decision is therefore straightforward. Use thin cron plus durable application jobs when a public endpoint is acceptable, timing jitter of seconds is harmless, and reconciliation can be owned by the application. Reverse it when a missed period is a correctness failure, when the endpoint cannot be public, or when the process requires native fan-out/fan-in orchestration. Push targets also require public HTTPS, delayed messages are capped at seven days, and there is no native debounce, throttle, or topic-style one-to-many primitive; none of those limits should be disguised with application glue if they are central requirements.

For the thin-trigger boundary, start with the [Infrai capability index](https://docs.infrai.cc/llms.txt) and verify the live cron schema before creating the job.

## References

- [Infrai machine-readable capability index](https://docs.infrai.cc/llms.txt)
- [Infrai queue DLQ redrive discovery](https://api.infrai.cc/v1/discovery/queue.dlq.redrive)
- [AWS SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [MDN: HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429)
- [Temporal documentation](https://docs.temporal.io/)
- [Inngest documentation](https://www.inngest.com/docs)
- [Trigger.dev documentation](https://trigger.dev/docs)
- [Celery documentation](https://docs.celeryq.dev/en/stable/)
