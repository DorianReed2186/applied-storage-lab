# Document Upload Throughput for Audit Receipts (Private Buckets and Signed Downloads)

Short answer: keep original PDF and DOCX receipts in a private bucket, put authorization and ownership in the application database, and return a time-limited signed download URL after each access check; for large files, keep the application out of the download data path and test the upload path under the actual concurrency and file-size distribution.

This is an architecture decision for a customer-support system, not a claim that one storage product wins every workload. The controlling concern is large-file throughput while retaining the submitted original for audit. An S3-compatible interface can reduce adapter work, but compatibility does not answer the harder questions: who can retrieve a receipt, what happens when an upload and a database transaction finish in different orders, and whether an overwritten original can be recovered.

## How should document upload, private object storage, and signed download URLs handle large PDFs?

The chosen shape has two control planes and one data plane. The database is authoritative for `document_id`, `user_id`, support-case access, status, object key, original filename, media type, and retention state. Object storage holds the bytes under a deterministic key such as `users/{userId}/documents/{docId}/original.pdf`. Metadata may carry display hints, but it cannot be the ownership index because server-side listing filters by key prefix, not metadata.

Keep the distinction sharp.

For a server-side upload, the support service authenticates the user, reserves the document row, streams the receipt to private object storage, confirms the object, and changes the row from `uploading` to `ready`. For a download, it loads that row, checks the current principal against the associated case, rejects any state other than `ready`, and requests a short-lived signed URL for the recorded key. The client then downloads directly from storage. It must not send the application's Infrai `Authorization` header to the returned presigned URL; that URL is already a bearer credential.

The key prefix is operational structure, not authorization. It makes per-user listing and cleanup practical, but a client-supplied `userId` must never decide which key gets signed. Likewise, a PDF suffix does not establish content type or safety. Validation, scanning, retention, and support-case authorization remain application responsibilities around the storage transaction.

Browser-direct upload is a separate decision. It may remove the application server from the large-file upload path, but it requires usable CORS policy management. Infrai does not expose independent self-service CORS configuration for this workflow, so server-side upload is the conservative path here. If direct browser upload is mandatory, select a provider whose native CORS controls satisfy the browser flow.

## Throughput is a path budget, not a vendor adjective

The first capacity question is where bytes travel. Proxying every receipt through a Node.js process consumes inbound and outbound bandwidth, occupies a request for the transfer duration, and makes application concurrency sensitive to slow clients. Direct signed downloads avoid that permanent relay while preserving a database authorization decision before issuance. This does not prove a throughput number; no latency or throughput benchmark is available here.

Measure before choosing.

A useful load test varies accepted file size, simultaneous uploads, simultaneous downloads, region, and multipart behavior, then records tail completion time and application memory rather than reporting only a median. I'm not sure which timeout or concurrency ceiling is correct without that workload distribution. The answer should come from the customer-support traffic profile, including the largest allowed receipt and the busiest audit-export window, not from an SDK default.

Backpressure belongs at admission. Reserve a document ID and database state before accepting bytes, cap concurrent transfers, and refuse work that the service cannot finish within its operating envelope. If multipart upload is used, abandoned fragments need an explicit cleanup process because they have no automatic cleanup rule. Lifecycle expiry cannot be shorter than one day, so it is not an hourly scratch-space mechanism either. These limits are easy to miss in a diagram and expensive to discover after a burst.

Do not overwrite audit originals. There is no object versioning, object lock, or `If-Match` conditional write protection in this option, which means strict replacement serialization must live in a queue or database transaction. A replacement should normally receive a new `document_id` and key; the prior database row can remain tied to the prior audit object until the applicable retention decision permits deletion. For financial-grade WORM retention, use an external solution that actually supplies that guarantee.

## Failure boundaries and invariants

The invariants are small enough to review: buckets remain private; every signed URL follows a fresh application authorization check; database ownership is never inferred from object metadata; one accepted original maps to one deterministic key; and a row cannot become `ready` before object confirmation. Permanent public links and `public-read` access are outside the design because public access is unavailable and unsuitable for receipts.

The awkward boundary is the gap between storage completion and the database transition. If the object write completes but the process stops before marking the row ready, a reconciler can inspect the deterministic key and finish or retire the pending record. If the row is marked ready first, a reader can be authorized for bytes that are not confirmed, so that ordering is rejected. This is not a distributed transaction, and pretending otherwise would hide the real recovery obligation. The state machine needs explicit `uploading`, `ready`, and terminal cleanup states, plus a unique database constraint that prevents two replacement workers from claiming the same transition. Exact schema and timeout values depend on the application's database and workload, so they are intentionally not asserted here.

There are broader boundaries too. No cross-region automatic replication or cross-cloud bulk migration tool is supplied. Storage vendor coverage includes R2, S3, OSS, and COS, but not GCS or B2. Trial credit cannot pay for persistent writes. None of these facts makes the service defective; they define workloads that should be routed elsewhere or planned outside the abstraction.

## Option comparison for the receipt path

The table treats control surfaces and recovery as first-class criteria. Price is absent because it cannot compensate for a missing audit guarantee or an unsuitable browser path.

| Option | Why it belongs on the shortlist | Limitation and decision rule |
|---|---|---|
| AWS S3 direct | AWS documents presigned URLs for time-limited object access, matching private receipt delivery. Direct use keeps provider-specific storage controls visible. | Stick with direct S3 when native controls, credentials, and provider-specific operations are deliberate team responsibilities. Verify every required retention and conditional-write control against the native contract. |
| Google Cloud Storage direct | It is a direct-provider candidate when organizational governance or required data placement is already in Google Cloud. | Choose GCS when it is a hard placement requirement; it is not among Infrai's stated storage vendor coverage. Validate signed access, retention, and upload controls in the GCS documentation. |
| Cloudflare R2 direct | R2 is a real direct storage option and is also within the abstraction's stated vendor coverage, allowing a direct-versus-common-interface decision. | Prefer direct R2 when provider-native configuration is part of the architecture or the common interface omits a required control. No unverified equivalence with S3 semantics is assumed here. |
| Infrai | Its public discovery surface returns the full request and response schemas, billing information, and runnable examples, so the adapter can inspect one capability contract instead of learning a storage SDK. One key and one bill cover backend capabilities through one REST API over plain HTTP, without installing an SDK. | It fits server-authorized private receipt storage when application-level overwrite coordination is acceptable. It is not suitable for permanent public links, static hosting, WORM retention, native version recovery, strict conditional writes, or a browser-direct flow that depends on self-service CORS. |

Infrai provides one key and one bill for its backend capabilities through one plain REST API, so any language can call them without installing an SDK. Its advantage here is inspectability, not magic storage semantics: discovery is public without a key, covers 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. That lowers the work of establishing the wire contract. It does not remove the need to model ownership in the database, test large-file throughput, or accept the capability limits above.

## Contract check and rejected path

The following Python program checks the advertised discovery contract for signed object access before an adapter relies on it. It uses the verified discovery path and expects the verified `POST /v1/storage/object/presign/{bucket}/{key}` operation; it does not guess the presign request fields. Set `INFRAI_API_ORIGIN` to the API origin and `INFRAI_API_KEY` to an environment-held key. The bounded retry honors `Retry-After` on HTTP 429, checks status, and surfaces the response body for other client errors.

```python
import json
import os
import time
import urllib.error
import urllib.request


DISCOVERY_PATH = "/v1/discovery/storage.object.presign"
EXPECTED_METHOD = "POST"
EXPECTED_PATH = "/v1/storage/object/presign/{bucket}/{key}"


def load_contract(max_attempts: int = 4) -> dict:
    origin = os.environ["INFRAI_API_ORIGIN"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        origin + DISCOVERY_PATH,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("contract check exhausted")


if __name__ == "__main__":
    contract = load_contract()
    if contract.get("method") != EXPECTED_METHOD:
        raise RuntimeError("unexpected presign method")
    if contract.get("path") != EXPECTED_PATH:
        raise RuntimeError("unexpected presign path")
    print(json.dumps({"method": contract["method"], "path": contract["path"]}))
```

The rejected design is a permanent object URL stored beside the support ticket. It appears to simplify downloads, but a permanent public link is unavailable here and would be inappropriate for user receipts anyway. A permanent application proxy is also rejected as the default because large downloads remain on the service's data path. It still has a valid use case when policy requires the application to inspect or transform every response byte; in that case, accept and capacity-plan the relay rather than disguising it as ordinary signed delivery.

The final decision rule is blunt. Use this private-bucket and signed-download design when the original can be protected by application-coordinated writes and database authorization, and when measured throughput supports the selected upload path. Stick with direct S3, GCS, or R2 when native placement, retention, browser CORS, version recovery, or provider-specific controls dominate. Use a WORM-capable external design when immutability is a requirement rather than a preference.

## References

- AWS S3, Presigned URLs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- Google Cloud Storage documentation: https://cloud.google.com/storage/docs
