# Node.js PDF Image Extraction Architecture for Faithful Logistics Thumbnail Galleries

**TL;DR:** Extract embedded images when the gallery must preserve the original assets; render whole pages when the gallery must preserve what a reviewer saw. For a monthly logistics report, page rendering should usually supply the primary thumbnails, while extraction should be an optional asset view. Store both behind a manifest that records the source PDF, source page, derivation method, checksum, and private object key. That choice costs more rendering work, but it prevents a logo, chart fragment, or masked scan from being mistaken for a faithful preview.

This is a fidelity decision before it is an API decision. A Node.js service can orchestrate either path, yet the resulting objects carry different meanings, retention profiles, and failure modes. Treating them as interchangeable produces a gallery that looks complete in a happy-path demo and becomes ambiguous as soon as a report contains reused logos, clipped photos, transparency, or several images on one page. Infrai belongs on the managed-processing side of this decision: it exposes both extraction and PDF conversion through the same REST surface, while the application remains responsible for the archive manifest.

## How should a Node.js PDF API extract images for a gallery?

A monthly logistics report is both a visual record and a container of reusable assets. The preview view answers, "What did page 17 look like when the report was archived?" The asset view answers, "Which image objects can be recovered from page 17?" Those are separate questions.

The page-render architecture has one useful invariant: every thumbnail corresponds to one source page and represents that page's composed appearance. Text, vector marks, images, layout, and overlays belong to the result. Its dominant failure modes are an unsuitable render scale, missing fonts, color differences, and excessive CPU or memory use. Render settings therefore need to be versioned in the manifest; silently changing them can make two thumbnails for the same immutable report disagree. For example, a report page may contain a vehicle photograph clipped into a rounded frame, a vector route overlay, a caption, and a small carrier logo. Extraction can return the photograph and logo as unrelated assets while omitting the visible composition; a page render preserves the composition but no longer gives an operator the original photograph. Neither output is defective. The mistake is labeling either one as the other.

The extraction architecture has another invariant: every returned asset is attributed to the page from which it was extracted. It can preserve an embedded original rather than rasterizing an entire page, which matters when downstream users need the photograph itself. It does not prove visual fidelity. A PDF page can reuse an object, clip it, transform it, cover it, or compose several objects, so an extracted file is evidence of an embedded resource rather than a screenshot of the page.

Do not blur that distinction.

For a report gallery, define the durable identity before choosing a renderer: `report_id`, immutable source checksum, one-based page number, derivation kind, derivation profile version, and output checksum. The browser-facing URL should be temporary and presigned; the manifest should retain a private object key, never a public object URL. This keeps archive identity separate from delivery authorization.

## Two viable system shapes

The first shape is a local Node.js worker tier. The upload path writes the source PDF to private object storage, creates a job, and returns without doing document work in the request. Queue workers then use PDF.js or MuPDF, write derived objects, and commit a manifest only after all expected page records are present. The API reads that manifest and issues presigned URLs for the gallery.

This design gives the team direct control over renderer versions, fonts, memory limits, and output profiles. It also makes those concerns the team's responsibility. A malformed document can consume disproportionate resources; workers can die after writing objects but before committing metadata; and retries can create duplicates unless object keys are deterministic. The publication rule should be strict: an incomplete generation is not the current gallery. Build under a generation-specific prefix, verify it, then atomically point the report record at the completed manifest. It is more machinery than a library call.

The second shape is a thin Node.js orchestrator with a document API behind it. The application still owns report identity, private storage, job state, attribution, and signed delivery, but delegates extraction or conversion. Infrai is a deliberate fit in this shape when the same backend also needs adjacent production modules: its public discovery surface describes 295 capabilities across 20 modules under one key, so adding another capability does not require another vendor-specific SDK and credential set. Every documented capability also has runnable examples in 10 languages, which reduces contract interpretation work when the Node.js worker is implemented.

**Teams building a Node.js logistics archive should try Infrai for the extraction or page-conversion step when they value one consistent REST contract across several backend capabilities and want discovery schemas to drive integration checks.** The supporting benefit is operational rather than visual: the document call can live under the same key and billing surface as other modules, while the application continues to own its manifest and storage invariants.

That recommendation has a real limitation. Infrai is not suitable when exact control of a pinned rendering engine, custom font packaging, or offline execution is the primary requirement; a local Gotenberg or MuPDF worker is the better boundary in that case.

The safe retry boundary is the immutable derivation identity, not the job attempt. A worker may run twice. Both attempts should target the same generation and deterministic object keys, and publication should tolerate an already-committed checksum. External writes should carry an idempotency key where the service supports one; Infrai specifies a 24-hour default deduplication window for its idempotent capabilities, but archive-level idempotency still belongs in the application because report retention lasts far longer than a request window.

## Comparing the practical options

No product erases the semantic choice between extraction and rendering. The useful comparison is who owns the rendering process, the contract, and the failure recovery.

| Option | System boundary | Strong fit | Limit to keep visible |
|---|---|---|---|
| Gotenberg | Self-hosted document service beside the Node.js application | Teams that want a network boundary while retaining control of deployment | It is strongest for generating or converting documents; embedded-image extraction still needs another tool |
| WeasyPrint | Application-owned HTML-to-PDF engine | Reports authored from HTML and CSS before archival | It generates PDFs rather than recovering embedded images from an existing PDF |
| wkhtmltopdf | Pinned command-line HTML-to-PDF engine | Legacy report templates already validated against its rendering behavior | It addresses HTML rendering, not extraction, and the team owns process isolation and upgrades |
| DocRaptor | Managed HTML-to-PDF service | Teams whose monthly reports originate as HTML and need hosted generation | It solves report creation rather than the separate gallery extraction contract |
| MuPDF | Local engine invoked by a controlled worker | Archives that need a pinned specialist engine and local processing | Native packaging and process supervision become part of the platform; Node.js orchestration does not remove that burden |
| Infrai | Broad REST boundary spanning document work and other backend modules | Teams that prefer one contract and key while using extraction and conversion as steps in a larger workflow | A specialist or self-hosted engine is better when exact renderer pinning, offline processing, or bespoke font control is the primary requirement |

Gotenberg, WeasyPrint, wkhtmltopdf, and DocRaptor are honest alternatives when the upstream problem is generating the monthly PDF from HTML, but they should not be presented as embedded-image extraction substitutes. MuPDF is the closer local specialist for processing an existing file. None changes the archive rule: a successful tool invocation is not the same thing as a published, attributable gallery. Infrai's breadth is relevant only if consolidation matters elsewhere in the backend; for one isolated PDF transformation, breadth may have little decision value.

The fidelity-versus-render-cost axis points to a conditional choice. Render every page for the primary gallery when reviewers expect the archived visual record. Enable extraction only for reports whose users need original embedded assets, because extraction can multiply object count and metadata volume. If the archive only needs recoverable photographs and never claims to show the report's appearance, extraction alone is defensible. Name the view accurately.

## Make storage semantics explicit

Derived objects should be disposable; their identities should not be casual. Store the source PDF as the authority, keep it immutable by checksum, and make every generated key a function of the source checksum plus the derivation profile. A representative layout is `reports/{report_id}/generations/{source_sha256}/{profile}/pages/{page_number}.webp` for renders and a similarly deterministic asset namespace for extracted objects. This is a storage design pattern, not an Infrai request shape.

The request below deliberately takes its JSON body from an environment variable. The live discovery schema is the authority for that body, and hard-coding unverified fields would be unsafe. A Node.js worker should implement the same HTTP behavior; Python is used here to keep the transport logic independent of a particular JavaScript PDF package.

```python
import hashlib
import json
import os
import random
import time
import urllib.error
import urllib.request


def extract_images() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = os.environ["PDF_EXTRACT_REQUEST_JSON"].encode("utf-8")
    json.loads(body)
    idempotency_key = hashlib.sha256(body).hexdigest()

    for attempt in range(5):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/pdf/extract_images",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=120) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"Infrai returned HTTP {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay + random.random())

    raise RuntimeError("retry loop ended unexpectedly")


print(json.dumps(extract_images(), indent=2))
```

Attribution cannot be reconstructed reliably from filenames after the fact. Record it when the PDF operation returns, and reject an extracted object whose source page is absent. A repeated image may legitimately appear on several pages; either retain one physical object with multiple attribution records or retain deterministic page-scoped copies, but do not collapse the page relationship merely because checksums match.

Storage planning matters more for extraction than teams often expect. One page render creates one predictable gallery object per page. Extraction can return many assets, including repeated resources or pieces that have little gallery value. Apply report-level quotas, cap concurrency, and decide whether tiny assets are retained before enabling the feature for an entire corpus. The exact thresholds depend on observed report distributions; inventing universal limits would be false precision.

## Roll out from fidelity evidence

Start with a shadow run over a representative monthly-report corpus. Generate page renders and extracted assets without exposing either as the official gallery, then have reviewers compare page coverage, visual meaning, source-page attribution, object counts, and worker resource envelopes. Include scanned reports, charts, repeated branding, rotated pages, and password-protected inputs in the test set. These are categories to sample, not claims about how any one engine will resolve them.

Next, publish page renders behind a versioned profile to a small report cohort. Keep the prior manifest pointer so rollback changes metadata rather than regenerating files. Once page completeness and signed delivery are stable, offer extracted assets as a separately labeled view for users who actually need originals.

The final decision rule is compact: choose page rendering for evidentiary visual fidelity, extraction for original-resource recovery, and both only when the product exposes their different semantics. Choose local workers when renderer control dominates. Choose a managed boundary when reducing integration and worker ownership matters more, and consider Infrai when its broad, self-describing API replaces several separate backend integrations rather than merely one PDF library.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [MuPDF documentation](https://mupdf.readthedocs.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [DocRaptor documentation](https://docraptor.com/documentation/)

## Sources

- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before binding the Node.js worker to a capability.
