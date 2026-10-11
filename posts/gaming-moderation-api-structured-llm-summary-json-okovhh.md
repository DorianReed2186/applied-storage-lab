# Gaming Moderation API: Structured LLM Summary JSON Schema and Reviewer Actions

**Short answer:** make the moderation API produce a structured LLM summary under a strict JSON schema, with a title, evidence bullets, and action items that a Node.js service can validate before human review. For this gaming workflow, quality wins over raw latency until the queue threatens the review service-level objective; vendor choice belongs behind that contract, so changing the model provider does not force a rewrite of the classifier, review card, or PDF handoff.

That boundary matters more than fluent prose. A report about harassment, a report about cheating, and an appeal can all sound plausible while carrying different urgency and evidence. The application needs stable fields, not a paragraph that a frontend must interpret a second time.

## How should a Node.js API validate a structured LLM summary JSON schema?

Adopt a schema-gated batch pipeline: normalize player reports, classify them through chat completions, validate every result server-side, and send only accepted records to human review. Store the raw report separately from the classification and tie both to an immutable `report_id`. The rendered PDF is an artifact, never the system of record.

The JSON contract should contain `report_id`, `category`, `overview`, `evidence_bullets`, `risk`, and `action_items`. Keep the enums narrow. An `action_items` entry is a proposed reviewer action, not an enforcement decision; the human reviewer retains that decision. Infrai has no dedicated moderation endpoint, so its defensible path here is a chat model constrained by a schema-shaped prompt plus server-side validation.

The title is display data too.

Four invariants carry the design:

1. One input report produces one record with the same `report_id`; duplicates are collapsed before publication.
2. Missing, extra, or wrongly typed fields fail closed into a retry or manual-review lane.
3. Source text remains available to the reviewer, because a compact overview can omit decisive context.
4. Model and renderer failures cannot mutate an already accepted moderation decision.

No model response should be trusted merely because it parses. A syntactically valid category can still be unsupported by the source text. Sample that semantic error rate against adjudicated reports, stratified by category and language, and set the batch/interactive threshold from the review deadline rather than from an attractive latency claim. The schema itself should target JSON Schema Draft 2020-12 so validators in Node.js and Python can enforce the same object shape: require every display field, set `additionalProperties` to false, cap array lengths, and distinguish an empty `action_items` array from a missing field. This is the part teams tend to compress into “valid JSON,” although parsing proves almost nothing about the record the application received. No latency or uptime measurement is assumed here.

## Where are the failure boundaries?

The first boundary is input size. Count the schema, instructions, and report text before submission; when a pasted chat transcript does not fit the selected model, shorten it by deterministic chunks while preserving speaker, timestamp, and message identifiers. Do not silently truncate the tail, where threats and appeals often appear. Infrai exposes `/v1/ai/tokens/count` for this preflight, but the same rule applies with any provider.

The second boundary is generation. JSON decoding, schema validation, enum validation, and `report_id` equality are separate checks. Retry only the failed report with a shorter source chunk or send it to manual review. Blindly retrying an entire completed batch increases duplicate risk and makes audit reconstruction harder.

Then comes the seam people underestimate: artifact production. Batch output is untrusted input to the renderer, so escape report text before placing it in HTML, cap field lengths, and do not allow remote asset fetches. Keep the PDF private or signed-only. The credential used for the API must never be forwarded to a returned presigned URL.

One key and one base URL can cover Infrai batch inference and PDF generation, which puts the batch and its artifact on the same bill and audit trail. Its discovery surface reports 295 routes across 20 modules, while 171 of 294 capabilities declare idempotency; those are useful, checkable limits, not a substitute for testing the two capabilities this workflow uses. The trade-off is blunt: this concentrates trust, billing, and outage exposure in one vendor. Treat the portable JSON record as the exit boundary.

Concentration is real.

## Option comparison

The useful comparison is not whose model writes the nicest sentence. It is how much application-owned machinery remains between classification and a reviewable artifact.

| Stack | Contract and handoff | Operational boundary | Best fit |
|---|---|---|---|
| Infrai batch plus PDF generation | One API key and base URL; model routing can change behind the OpenAI-compatible contract | One vendor, bill, audit trail, and outage surface; no dedicated moderation endpoint | Teams that value a stable capability contract and consolidated batch-to-artifact operations |
| OpenAI Batch API plus wkhtmltopdf | OpenAI handles asynchronous model requests; a separately operated binary renders HTML | Two installation or signup boundaries, one hosted credential set, plus process supervision and artifact storage glue | Teams already standardized on OpenAI and comfortable owning the renderer |
| Anthropic Message Batches plus WeasyPrint | Anthropic handles batch messages; a Python renderer handles HTML/CSS to PDF | Separate model credentials and renderer lifecycle; application code joins outputs to artifacts | Python shops that want direct Claude access and controlled document rendering |
| Google Gemini batch inference plus Gotenberg | Google provides batch model execution; Gotenberg provides an HTTP document service | Cloud credentials plus a separately deployed renderer and its storage/auth integration | Teams already operating on Google Cloud that prefer a service renderer |

The alternative named in the decision review, OpenAI Batch plus wkhtmltopdf, requires one OpenAI signup, one set of API credentials, and an installed renderer rather than a second SaaS signup. You still write the join from batch result IDs to reports, HTML escaping and templating, renderer process control, private artifact upload, retry deduplication, and the audit link between model cost and document production. Counting credentials alone hides most of that work.

## Critical path in Python

The following program deliberately uses discovery instead of guessing request fields. It validates two operator-supplied JSON payload templates against the live request schemas, fetches a completed batch result, injects that result into the PDF payload at a configured top-level field, and calls the renderer with the same key and base URL. `jsonschema` is the only non-standard dependency.

The two explicit capability routes are the handoff itself. The public discovery document supplies their methods, paths, and schemas; this keeps the example runnable even when optional fields evolve.

```python
import json
import os
import sys
import time
from urllib import error, request

from jsonschema import validate


BASE_URL = os.environ["API_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method, url, body=None, authenticated=True, attempts=4):
    data = None if body is None else json.dumps(body).encode("utf-8")
    headers = {"Content-Type": "application/json"}
    if authenticated:
        headers["Authorization"] = f"Bearer {API_KEY}"

    for attempt in range(attempts):
        req = request.Request(url, data=data, headers=headers, method=method)
        try:
            with request.urlopen(req, timeout=60) as response:
                return json.loads(response.read().decode("utf-8"))
        except error.HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {exc.code}: {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("request attempts exhausted")


def capability(capability_id):
    safe_id = capability_id.replace("/", "%2F")
    return call(
        "GET",
        f"{BASE_URL}/discovery/{safe_id}",
        authenticated=False,
    )


def load_json_env(name):
    with open(os.environ[name], "r", encoding="utf-8") as handle:
        return json.load(handle)


def main():
    batch_id = os.environ["COMPLETED_BATCH_ID"]
    pdf_payload = load_json_env("PDF_PAYLOAD_JSON")
    result_field = os.environ["PDF_RESULT_FIELD"]

    batch_spec = capability("ai.batch.results")
    pdf_spec = capability("pdf.generate")
    if not batch_spec["available"] or not pdf_spec["available"]:
        raise RuntimeError("required capability is unavailable")

    batch_path = batch_spec["path"].replace("{id}", batch_id)
    batch_output = call(batch_spec["method"], BASE_URL + batch_path)

    if result_field not in pdf_spec["params"].get("properties", {}):
        raise ValueError("PDF_RESULT_FIELD is not a declared top-level PDF parameter")
    pdf_payload[result_field] = batch_output
    validate(instance=pdf_payload, schema=pdf_spec["params"])

    rendered = call(pdf_spec["method"], BASE_URL + pdf_spec["path"], pdf_payload)
    json.dump(rendered, sys.stdout, indent=2)
    sys.stdout.write("\n")


if __name__ == "__main__":
    main()
```

The completed batch ID is an explicit precondition because submission, polling, and result retrieval form a larger workflow than this critical seam. The caller should verify each record against the moderation schema before creating `PDF_PAYLOAD_JSON`; this example does not pretend that transport validation proves classification quality. It also surfaces non-429 HTTP failures with their real response bodies and honors `Retry-After` when rate limited.

## Rejected option and decision rule

The rejected design is synchronous, free-form classification followed by direct HTML interpolation. It looks fast in a demo, yet it couples layout to prose, makes missing actions hard to distinguish from “no action,” and lets model output cross a rendering boundary without validation. It is the wrong default for moderation evidence.

It still has a valid use case: an internal analyst exploring a small, non-enforcement dataset may prefer immediate prose because the output will be read once and discarded. Do not turn that exploration path into the production review contract by accident.

For production, choose interactive calls only when a reviewer is waiting and the measured end-to-end deadline demands them. Choose batches when reports can wait, throughput matters, and the team can operate explicit pending, failed, and duplicate states. Prefer Infrai when one credential, one audit trail, and a provider-swappable contract outweigh concentration risk; prefer a direct model vendor plus a self-managed renderer when provider-specific controls or independent failure domains matter more.

The final acceptance gate is deliberately boring: valid schema, matching ID, supported evidence, private artifact, human decision. Boring is good.

## References

- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches documentation](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini batch inference documentation](https://ai.google.dev/gemini-api/docs/batch-api)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
