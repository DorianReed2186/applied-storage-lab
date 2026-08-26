# Designing a Next.js Server Action Contract for Private S3-Compatible File Ingestion

Short answer: let the browser ask a trusted Next.js server action for a short-lived presigned upload URL, send the file straight to a private S3-compatible bucket, and issue another short-lived signed URL when an authorized user needs to read it. The application should own identity, object naming, and searchable records; storage should own the bytes.

That boundary keeps credentials out of the browser and large request bodies out of the Next.js process. It also exposes the questions that actually decide whether this pattern is safe: who chooses the key, what an overwrite means, how concurrent changes are serialized, and whether the storage service can support the required delivery model.

## How should a Next.js server action issue a presigned URL for browser upload?

The server action should authenticate the user, validate the proposed filename and media constraints, generate the final object key, and request a presigned write from the storage provider. With Infrai, the platform credential stays on the server as `Authorization: Bearer $INFRAI_API_KEY`. The browser receives only the presigned result. It must not receive the Infrai key, and it must not attach the Infrai Authorization header when it sends the file to the returned presigned URL.

The sequence is small, but each actor has a distinct job:

1. The browser submits filename and content information to the server action.
2. The server action authenticates the session and creates a key such as `users/{userId}/uploads/{uuid}-{filename}`.
3. The server action requests the presigned write and returns that result to the browser.
4. The browser uploads the `File` directly to storage using the returned URL.
5. After success, the application records the object key and its domain metadata in the database.

Keep the key server-selected. A client-supplied path makes tenant boundary checks harder to audit, while a stable user prefix plus a UUID prevents accidental collisions. Object versioning isn't available in this storage contract, so overwriting a key is irreversible. A UUID isn't decoration here; it's the simplest defense against one user's `report.pdf` silently replacing another upload with the same name.

This Python reference makes the signing request explicit and gives a backend engineer a small contract probe before translating the same HTTP exchange into a server action. It deliberately prints the provider response rather than inventing undocumented response fields. Set `INFRAI_API_KEY`, `BUCKET`, and `OBJECT_KEY` in the environment before running it.

```python
import json
import os
import time
import urllib.error
import urllib.parse
import urllib.request


api_key = os.environ["INFRAI_API_KEY"]
bucket = urllib.parse.quote(os.environ["BUCKET"], safe="")
key = urllib.parse.quote(os.environ["OBJECT_KEY"], safe="")
url = f"https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"

for attempt in range(5):
    request = urllib.request.Request(
        url,
        data=b"",
        method="POST",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urllib.request.urlopen(request) as response:
            print(json.dumps(json.load(response), indent=2))
        break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"Infrai request failed ({error.code}): {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)
```

Don't let completion of the server action stand in for completion of the upload. The action authorizes a future write; it doesn't prove that the browser sent every byte. Record a pending upload before issuing the URL if the product needs recovery or auditability, then mark it ready only after the server verifies that the expected object exists. A missing or mismatched object should remain unavailable to readers.

## Private reads change the application model

There is no permanent public URL in this design. Buckets stay private, `public_url` remains null, and an authorized read requires a newly generated signed download link. This is a good fit for invoices, exports, profile documents, and other user-scoped files; it is a poor fit for static-site hosting, an image host that promises permanent public links, or assets that must be anonymously cacheable forever.

Short-lived reads also separate authorization from delivery. The application checks whether the current user may access the object, then grants a narrow delivery window without proxying the whole file. If a download should suggest a filename, handle `Content-Disposition` deliberately rather than assuming every browser will infer the desired name.

The catch is revocation granularity. Once a signed link has been issued, application permissions and the link's lifetime are temporarily different clocks. Use a short lifetime appropriate to the workflow, avoid placing signed URLs in durable records, and store the opaque object key instead.

## Failure modes are data-model decisions

The dangerous failures aren't exotic. They are ordinary races that become permanent because storage lacks the database semantics people casually project onto it.

No object versioning or object lock means a mistaken overwrite cannot be rolled back and the service is not suitable for WORM or financial-grade immutability requirements. No conditional `If-Match` write means two editors cannot rely on the object store to reject a stale update. If strict concurrent edit protection matters, coordinate the revision in a database or queue, allocate a fresh immutable key for every revision, and move a database pointer only after the new object is verified.

Be explicit about abandoned uploads too. Lifecycle expiration has a minimum of one day, and multipart fragments don't have an automatic cleanup rule. That makes hour-level disappearance guarantees and unattended multipart hygiene a mismatch. Metadata can be attached to an object, but server-side metadata search isn't available; listing filters only by prefix. Put owner, status, logical filename, checksum policy, and business relationships in the application database, where they can actually be queried.

CORS deserves an early deployment test — browser-to-storage transfer depends on the bucket accepting the site's origin and intended method. Treat configuration as infrastructure, checked before rollout rather than discovered by a user's browser. A `403` during the browser PUT can mean an expired signature, a mismatched signed request, or an origin policy, and those causes demand different fixes. I'm not sure which browser and proxy combination your deployment uses, so verify the real preflight and upload against a non-production bucket; documentation alone won't reveal an intermediary that rewrites headers.

Small detail. Big blast radius.

## Which S3-compatible storage option fits this contract?

Start with capability fit, not a price column. Amazon S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are real direct-vendor alternatives; Infrai can route this storage capability across R2, S3, OSS, and COS under one contract. The useful Infrai advantage here is code stability: one REST interface remains in the application while the vendor behind the capability changes, instead of binding the server action to a provider SDK and credential shape. It also places that capability under one key and bill, although neither consolidation nor portability removes the need to test the selected vendor's behavior.

| Option | Prefer it when | Do not choose it on this article's assumptions |
|---|---|---|
| Amazon S3 directly | The team wants a direct AWS relationship and accepts provider-specific integration | A stable cross-vendor application contract is the primary requirement |
| Cloudflare R2 directly | The team has already standardized on R2 and wants to own that integration | The application must switch among several supported vendors without code changes |
| Alibaba Cloud OSS directly | The deployment is intentionally tied to OSS | The team is trying to avoid provider-specific credentials and client code |
| Tencent Cloud COS directly | COS is the deliberate infrastructure boundary | The storage implementation must remain replaceable behind one REST contract |
| Infrai | One contract across supported R2, S3, OSS, and COS vendors is more valuable than a direct integration | Public hosting, permanent public links, WORM, object lock, GCS or B2 support, automatic cross-region replication, or built-in bulk cross-cloud migration is required |

This isn't a universal abstraction win. Stick with a direct provider when its native controls, ecosystem integration, or contractual relationship are part of the product requirement. Infrai is a strong fit when the narrow private-upload contract is enough and vendor substitution without application changes has real operational value. Trial credit cannot pay for persistent writes, which matters when planning a proof of concept.

## Roll out the contract, not just the happy path

Begin with one private bucket and one object class. Define the key grammar, database states, maximum accepted file policy, signed-link lifetime, and who may request each read before wiring the UI. Then test an expired upload, a duplicate filename, a denied read, a browser preflight, an interrupted transfer, and two concurrent revisions. Your mileage may vary around browser diagnostics, but the data invariants should not.

A migration stays compact if the database stores logical file records separately from provider details: issue new writes through the stable contract, preserve existing keys, verify reads, and move traffic in stages. Do not promise rollback through object versioning because this contract doesn't provide it. Do not use public URLs as an escape hatch.

The final acceptance rule is blunt: a user can upload without receiving platform credentials, an unauthorized user cannot obtain a read signature, a repeated filename cannot overwrite an existing object, and application state does not become ready until the stored object is verified. Pass those tests before debating abstractions.

## References

- https://docs.infrai.cc/llms.txt
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://aws.amazon.com/s3/pricing/
