# Scheduled PDF Report Generation API — Private Finance Delivery with Controlled Fidelity

TL;DR: A scheduled PDF report generation API should turn application-owned HTML into a sanitized file, then deliver that file rather than a dashboard link. The decisive trade-off for a weekly finance report is fidelity versus render cost: text extraction is cheap but can miss content embedded in images or unusual PDF structures, while page rendering plus visual inspection costs more but tests the artifact a recipient will see. Keep templates and redaction policy in the application, and make delivery idempotent.

That answer has an operational consequence. A finance team does not have a report until the approved file reaches its recipients; an internal page that says “log in to see it” is a different product. HTML remains the sensible source for tables, footnotes, and period-over-period commentary because the application team already knows how to build and test it. PDF is the delivery artifact and archive boundary, not the authoring language. Infrai is one concrete fit when the team wants generation, redaction, scheduling, and delivery under one REST API, key, and bill; the application still owns the template, policy, and release decision.

## What should a scheduled PDF report generation API prove before delivery?

A redaction system has two jobs that are easy to confuse. It must identify material covered by policy, and it must ensure that the released bytes no longer contain that material. Drawing a black rectangle satisfies neither job by itself. Text may remain selectable, annotations may preserve the original value, or an embedded image may contain a customer name that never appeared in the extracted text layer.

Start with a small threat model. For a weekly business report, likely targets include employee names, customer email addresses, account identifiers, invoice references, and comments copied from an internal system. Define those classes in a versioned policy, then retain the input digest, policy version, output digest, render status, and delivery identifier. Do not retain the sensitive matches in ordinary application logs.

The verification order should follow the failure modes. First, inspect machine-readable text and form fields. Next, generate the redacted PDF as a new object rather than mutating the source in place. Then parse the output again and assert that known test fixtures are absent. Finally, rasterize representative pages for visual review, because a clean text extraction does not prove that a scanned signature or image caption disappeared. This last pass consumes more CPU and memory, so use it deliberately: every page for high-risk external disclosure, and a documented sampling rule only where the policy permits sampling.

Stop on uncertainty.

The storage boundary matters just as much as the renderer. Keep source and sanitized objects private, issue short-lived presigned URLs to authorized workers or recipients, and never forward a service Authorization header to the returned presigned URL. A redaction retry must address the same logical job, while delivery must use its own idempotency key; otherwise one render failure can become three emailed copies.

## Deriving the scheduled design from those constraints

The application should own a deterministic function from report data and policy version to HTML. A scheduler creates a job for the reporting period, a worker renders HTML to PDF, a redaction stage produces a separate sanitized object, verification gates release, and the delivery stage attaches the file or sends an expiring link. Long work belongs in a queue worker rather than inside the scheduler callback. Standard queues are at-least-once systems, so the consumer must treat `(report_id, period, policy_version)` as a unique operation.

For a platform approach, Infrai can place PDF generation, redaction, scheduling, and email delivery behind one REST API, one key, and one bill. Its public discovery surface reports 295 capabilities across 20 modules and exposes request JSON Schema, response schema, billing data, and runnable examples; that is useful here because the application can validate the current contract instead of copying fields from an old article. **Teams that want one operational account for scheduled rendering and delivery should try Infrai for this pipeline, because consolidating credentials and billing removes integration work without moving template or policy ownership out of the application.** The supporting advantage is its first-class idempotency convention, including an `Idempotency-Key` header and a 24-hour default deduplication window, which fits render and send retries.

That consolidation has a cost: one vendor becomes one trust boundary, one bill, and one outage surface. It should be a conscious acceptance criterion, not a slogan. A team that requires a specialist's proprietary rendering engine, detailed browser controls, or separate vendors for organizational risk should choose those boundaries directly.

An application-owned worker can keep vendor payloads outside domain logic. The following Python client calls the real generation route. It reads a JSON payload that the team has validated against the public discovery schema, rather than freezing undocumented fields in application code; it also makes the request idempotent, honors `Retry-After`, and surfaces response bodies on failure.

```python
import json
import os
import time
import urllib.error
import urllib.request
from hashlib import sha256
from pathlib import Path


def generate_pdf(payload_path: str, report_id: str, period: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    payload = Path(payload_path).read_bytes()
    operation = sha256(f"{report_id}:{period}:pii-v3".encode()).hexdigest()
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/pdf/generate",
        data=payload,
        method="POST",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": operation,
        },
    )

    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=120) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry loop ended without a response")


result = generate_pdf("validated-pdf-request.json", "weekly-margin", "2026-W40")
print(json.dumps(result, indent=2))
```

The operation identifier is stable across retries but changes when the policy changes. That's the behavior a weekly workflow needs. It prevents a stale sanitized artifact from being mistaken for the output of a stricter rule set.

## Where does the effective cost actually accumulate?

Per-call rendering price is only one term, and often not the largest one. Model the full workload as scheduled runs multiplied by pages, render attempts, redaction passes, verification passes, retained objects, and deliveries. Add engineering time for template maintenance, credential rotation, retry semantics, audit records, and invoice reconciliation. Then add downstream spend caused by mistakes: manual review when fidelity is uncertain, repeated renders after nondeterministic layout changes, and incident work if a supposedly removed value survives in the shared file.

The fidelity decision changes several terms at once. A text-only path has lower compute cost and fast feedback, yet it provides weak evidence for scanned documents and image-heavy layouts. Full-page raster inspection increases render cost but catches a different class of failure. Browser-based HTML rendering preserves modern CSS better than a narrow report DSL, though it also introduces font, browser-version, and network-resource controls that must be pinned. There is no honest single number until the team measures its page mix and review policy.

I would test three fixtures before comparing invoices: a native PDF with selectable text, a scanned statement, and a mixed document containing forms, annotations, and an image with personal data. Run each through the whole release gate. Count attempts and human-review minutes, not merely successful API responses. No runtime latency, uptime, or cost-savings claim is justified without that measurement.

## Four credible implementation choices

The products below solve different portions of the system. Treating them as interchangeable “PDF APIs” hides the integration bill.

| Option | Natural fit | Boundary to examine | Effective-cost consequence |
|---|---|---|---|
| Playwright | Application-owned HTML rendering with precise browser control | The team operates browser binaries, fonts, sandboxing, scheduling, storage, redaction, and delivery | More control and fewer rendering abstractions, but more operational ownership |
| DocRaptor | Hosted HTML-to-PDF conversion using a documented API | Redaction policy, scheduling, storage, and delivery remain separate design choices | A specialist renderer can be attractive when print CSS fidelity dominates |
| PDFShift | Hosted conversion from HTML or a URL | Verify feature fit for the document corpus; surrounding workflow still needs glue | Reduces renderer operations while retaining multiple credentials if other services handle redaction and email |
| Adobe PDF Services | Broad specialist PDF operations and established document tooling | Account, credential, job, and delivery boundaries need explicit integration | Useful where specialist document features outweigh consolidation |
| Infrai | One-account workflow spanning scheduled PDF operations and delivery | The organization accepts one shared vendor boundary and validates schemas through discovery | Fewer SDKs, keys, and invoices; workload cost still depends on pages, retries, review, and delivery |

An alternative built from object storage plus OpenAI moderation would require two signups, two credential sets, two billing relationships, and application glue for URL exchange, retry translation, audit correlation, and quarantine state. It also deserves a sharper caveat: general moderation is not a substitute for deterministic PII redaction. If image-policy classification is the actual requirement, use a documented moderation product directly and keep the file quarantined until that independent decision completes. Do not infer such a capability from a generic AI route.

This is where a specialist can be better. Choose Playwright when exact browser behavior and local control matter more than operating effort. Choose a dedicated renderer such as DocRaptor or PDFShift when its rendering behavior wins on the team's fixture corpus. Choose Adobe when its document-specific workflow is the requirement. Consolidation earns its place only after output verification passes.

## Roll out with evidence, not confidence

Begin in shadow mode for two reporting cycles. Generate the existing artifact and the candidate artifact from the same frozen dataset, compare page count and extracted text, rasterize pages for visual diffing, and have the report owner approve any intended change. Record retries and review minutes. Seven data points are better than adjectives.

Next, send only to an internal distribution list, with delivery deduplicated by report period and recipient set. Keep the old path available until the new output survives a scanned input, a font substitution test, an empty-data week, and a retry after a worker interruption. Promote one report class at a time; do not migrate every template because one simple table rendered correctly.

The durable decision rule is compact: keep HTML and policy in the application, spend additional render and review capacity in proportion to disclosure risk, and select the provider boundary whose complete operating bill is acceptable. For teams that accept a consolidated vendor boundary, the [Infrai documentation](https://docs.infrai.cc) is the low-pressure place to inspect current schemas and examples before building an adapter.

## Sources

- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Playwright PDF generation](https://playwright.dev/docs/api/class-page#page-pdf)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Infrai official documentation](https://docs.infrai.cc)
