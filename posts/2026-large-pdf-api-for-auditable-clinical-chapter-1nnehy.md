# 2026 Large PDF API for Auditable Clinical Chapter Delivery

TL;DR: For per-chapter delivery of a large clinical PDF, treat each chapter as a derived, signed artifact rather than as a byte-range copy. The API should accept explicit inclusive page ranges, produce a manifest that binds each output digest to its source digest and redaction policy, and publish a chapter only after extraction, redaction, validation, and receipt signing have all succeeded. In a Node.js service, keep those stages behind a job boundary so a retry cannot silently create a different disclosure.

The first design question is storage, because the bill is made of source bytes, derived chapter bytes, temporary working bytes, request and retrieval operations, and the audit records retained beside them. A useful model is `B_source + B_chapters + B_work + B_evidence`, multiplied by each class's retention window. For a dossier split into `C` chapters and retried `R` times, careless implementations let the temporary term approach `B_source * R` while also retaining every superseded output. The dominant term is therefore often duplicated document bytes, not the tiny signed manifest. Change that term first: stream one immutable source through bounded work space, retain only committed chapter objects, and expire failed intermediate objects promptly.

This is deliberate deletion. Keeping the source, the released chapters, and compact evidence preserves the material needed to verify what was disclosed; deleting temporary rasterizations and rejected drafts means a later investigation cannot inspect every transient representation. Record that trade-off in the retention policy instead of pretending storage is free or forensic reconstruction is unlimited.

Keep fewer bytes.

## How should an API split large PDF page ranges?

A page-range API that returns five files has answered the easy question. A healthtech workflow must answer the harder one: which source produced chapter 03, which pages were selected, which redaction rules ran, whether validation passed, and whether the bytes delivered later are the same bytes that passed review.

Make the source immutable for the lifetime of a job. Address it by a cryptographic digest, then normalize requested ranges into an ordered manifest such as pages 37 through 52, inclusive. Reject overlaps unless duplication is an explicit requirement; reject gaps when the request claims to partition the full dossier; and reject a range whose last page exceeds the parsed page count. Page numbering needs one convention at the boundary. Human labels commonly start at 1, while libraries may index from 0, so the adapter should convert once and preserve the submitted form in evidence. One off-by-one error can disclose the first page of the next patient's chapter.

The redaction stage must operate on each extracted artifact before release. Removing visible text is not enough as a design criterion: validation should inspect the serialized result for the prohibited identifiers defined by the policy, and the workflow should avoid carrying source metadata into outputs unless it is required. The claim in the receipt should stay narrow: this policy and validator accepted these exact bytes. It should not claim that no personal data exists anywhere in the file unless the system can actually establish that proposition.

No receipt, no release.

A compact evidence record can be signed after validation:

```python
from dataclasses import dataclass, asdict
from hashlib import sha256
import json

@dataclass(frozen=True)
class ChapterReceipt:
    job_id: str
    source_sha256: str
    chapter: str
    first_page: int
    last_page: int
    policy_version: str
    output_sha256: str
    validation_result: str

def canonical_bytes(receipt: ChapterReceipt) -> bytes:
    return json.dumps(
        asdict(receipt),
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=True,
    ).encode("ascii")

def digest(data: bytes) -> str:
    return sha256(data).hexdigest()
```

This example intentionally stops before choosing a signing library or key service. Canonical serialization, key identifiers, signature algorithm, certificate handling, and verification policy must be one reviewed profile; mixing defaults from unrelated libraries creates receipts that cannot be reproduced. The output digest is computed after every PDF mutation, including redaction and metadata cleanup. Sign the receipt containing that digest, not a temporary file created earlier in the pipeline.

## The API boundary should describe work, not library calls

The public contract needs a submission resource, job status, and a committed result manifest; it does not need to expose every parser operation. A Node.js handler can validate the request, stream the source into immutable object storage, enqueue work, and return a stable job identifier. Workers may use another runtime, which is useful here because the code path that parses hostile documents deserves separate memory and time limits.

Use idempotency at submission. Bind the idempotency key to the source digest, normalized page ranges, redaction policy version, and delivery policy. Reusing the same key with different inputs must be a conflict, not an invitation to overwrite the original audit chain. A retry with identical inputs should resolve to the same logical job and, once committed, the same manifest.

There are two reasonable response modes. Small, bounded documents may finish synchronously, but large dossiers fit an asynchronous job much better because parsing, redaction, and validation have variable duration. Do not hold an HTTP connection open merely to hide that variability. The final manifest should list each chapter's inclusive range, byte digest, media type, size, receipt reference, and delivery state. Return short-lived delivery authorization separately from durable evidence so expiring access does not mutate the audit record.

The implementation must also define backpressure. Limit source size, page count, range count, decompressed working size, and concurrent workers; the exact limits belong to capacity tests, not marketing copy. Queue admission should fail clearly before expensive processing begins. A parser timeout is a terminal processing failure unless the retry policy identifies a transient dependency failure. Blind retries against malformed input merely reproduce load.

## Failure modes shape the architecture

| Failure mode | Detection point | Required outcome | Evidence to retain |
|---|---|---|---|
| Range crosses a chapter boundary | Request validation or reviewer check | Reject before publication | Submitted and normalized ranges |
| Source changes during processing | Digest comparison | Abort the job | Expected and observed source digests |
| Redaction validation fails | Post-serialization scan | Quarantine every affected artifact | Policy version and failure category |
| Worker stops after uploading two of five chapters | Commit barrier | Publish none of the new set | Attempt identifier and object digests |
| Delivery object differs from reviewed output | Read-time digest check | Deny delivery and alert | Expected and observed output digests |
| Signing key is unavailable | Receipt stage | Leave result uncommitted | Key identifier and failure category |

The commit barrier matters. Uploading chapter 01 does not make it releasable; only an atomic manifest transition should expose a complete approved set. Object storage and a database rarely share a transaction, so write content-addressed objects first, then commit the manifest that references them. Unreferenced objects are garbage-collection candidates after a safety interval. This ordering prefers harmless orphaned bytes over a manifest that points at missing evidence.

Partial retry needs the same discipline. Reprocessing chapter 04 is acceptable when its input tuple is unchanged, but a policy revision creates a new result set, even if four output digests happen to match. Preserve lineage between sets. Do not edit a signed receipt in place.

That distinction matters.

Observability should follow the state machine: accepted, source verified, extracted, redacted, validated, signed, committed, and delivered. Metrics need counts and duration by stage, plus bounded failure categories; logs should carry job and attempt identifiers but no extracted clinical text. Audit events and operational logs have different retention and access needs. Treating them as one stream either keeps sensitive diagnostics too long or discards evidence too early.

## Choosing the mechanism without choosing a vendor

Evaluate candidates with a fixed corpus, including encrypted inputs, malformed cross-references, rotated pages, annotation-heavy files, mixed page sizes, embedded files, and documents near each declared limit. PDF is a structured format, so a visual spot check alone is weak evidence. Compare parsed page count, selected-page order, retained interactive content, metadata behavior, output reopenability, and deterministic digest behavior under identical inputs.

The signature decision deserves its own test. Verify receipts outside the production service, rotate the signing key in a staging exercise, and prove that an old receipt remains verifiable under the retention policy. Test tampering separately: change one range, one policy version, and one output byte. Each change should fail verification for an understandable reason.

A selection scorecard should weight the properties that govern disclosure risk, not feature volume:

- extraction fidelity on the organization's corpus;
- precise control over metadata, annotations, attachments, and forms;
- reproducible output or a documented reason output bytes vary;
- isolation controls for CPU, memory, temporary storage, and time;
- an independent validation path;
- support for externally verifiable signatures and durable key history;
- clear failure semantics and idempotent retries.

Cost belongs in the scorecard, but as measured resource shape: peak working bytes per source byte, output amplification, CPU time per page class, request volume, and retention growth. A low per-request price says little if the mechanism keeps three full temporary copies or forces indefinite retention of rejected artifacts. Measure with the real corpus and publish the assumptions beside the result.

## Retention is part of correctness

Define separate retention classes for immutable sources, approved chapters, signed receipts, access events, and temporary work. The evidence object is small enough to retain longer in many designs, while source and output retention should follow the clinical and legal purpose established by the organization. Cryptographic deletion claims also require care: deleting a data key can make content inaccessible, but the audit record must say what was deleted, under which policy, and when.

The final decision rule is strict: choose an approach only if it can bind page selection, redaction policy, output bytes, and delivery state into independently verifiable evidence while enforcing bounded work space. If it can split pages but cannot prove the disclosure, it is a file utility, not a sufficient healthtech delivery boundary. If it retains every intermediate forever, it has moved the risk rather than solved it.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
