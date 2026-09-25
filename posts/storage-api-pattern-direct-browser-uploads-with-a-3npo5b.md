# Storage API Pattern: Direct Browser Uploads With Auth and Validation

The operational constraint that changes this design is the deletion deadline. A signed document is not ready merely because a browser received a successful upload response; the application needs evidence that the expected private object exists, plus an authoritative database record saying when it must disappear. **Short answer:** authenticate in the backend, allocate the object key there, issue a short-lived presigned upload, and verify the object with `HEAD` before changing the document state to ready. Add storage notifications only when asynchronous processing is actually required.

For a media company retaining signed releases, I would make the database record the retention authority and storage the byte store. The record should move through `pending_upload`, `ready`, and `deleted`, with `delete_after` set by policy rather than supplied by the browser. This is the boring design. Good.

## Incident lesson: an upload callback is not evidence

Consider a bounded incident exercise, not a customer story: a client asks for an upload URL, loses its connection after sending the bytes, and retries the application callback. Meanwhile, a second tab repeats the flow for the same release. If the browser chooses the key, both attempts can target one object. If the callback alone marks the release ready, the database can claim success without checking storage. If deletion depends only on a bucket lifecycle rule, an application policy measured from signature acceptance can drift from an object policy measured from creation time.

The invariant is stricter: one authenticated document record owns one server-selected, unguessable object key; readiness follows an object verification; and deletion is driven by an idempotent application job that can prove what it attempted. Notifications may wake downstream processing, but they do not become the retention ledger.

Callbacks lie.

I initially find event-driven diagrams attractive because every arrow appears independent. Under an SLO review, they are harder to defend: there are more queues to drain, more duplicate deliveries to absorb, and another delay between the legal deadline and observable deletion. I would set two separate objectives instead: an upload-readiness objective covering presign through verification, and a deletion objective covering `delete_after` through confirmed absence. Capacity planning then starts with peak concurrent uploads, verification request rate, and the number of documents crossing a deadline in the largest scheduling bucket, not average daily bytes.

## How should a storage API validate direct browser uploads?

The API should accept a document identifier, authenticate and authorize the caller, look up the server-owned key, and request a private presigned upload. In a Node and Express application, this auth validation belongs in the server route before presigning; the same boundary applies to the Go example below. The browser uploads directly to the returned URL and must not attach the backend's bearer token to that storage request. It then asks the application to finalize the document; the application performs `HEAD` and checks the returned object against the expected record before committing `ready`. A webhook is useful for asynchronous processing, but it is not stronger evidence than that object check.

This Go program shows the backend half using the two verified routes. It generates no path from user input, handles rate limiting, surfaces error bodies, and keeps the provider credential server-side. The idempotency convention has a 24-hour default deduplication window; application state still guards retries outside it.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "time"
)

type client struct {
    key string
    base string
    http *http.Client
}

func (c client) call(ctx context.Context, method, path, idem string, body []byte) (json.RawMessage, error) {
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, method, c.base+path, bytes.NewReader(body))
        if err != nil { return nil, err }
        req.Header.Set("Authorization", "Bearer "+c.key)
        req.Header.Set("Content-Type", "application/json")
        if idem != "" { req.Header.Set("Idempotency-Key", idem) }

        resp, err := c.http.Do(req)
        if err != nil { return nil, err }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return nil, readErr }
        if resp.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
                delay = time.Duration(seconds) * time.Second
            }
            select {
            case <-ctx.Done(): return nil, ctx.Err()
            case <-time.After(delay): continue
            }
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return nil, fmt.Errorf("storage request failed: status=%d body=%s", resp.StatusCode, strings.TrimSpace(string(data)))
        }
        return json.RawMessage(data), nil
    }
    return nil, fmt.Errorf("storage request remained rate limited")
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" { panic("INFRAI_API_KEY is required") }
    base := os.Getenv("STORAGE_API_BASE_URL")
    if base == "" { panic("STORAGE_API_BASE_URL is required") }
    c := client{key: key, base: base, http: &http.Client{Timeout: 15 * time.Second}}
    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()

    bucket := "signed-releases"
    objectKey := "tenant-42/documents/018f6f3a.pdf" // Allocated by the backend.
    escapedKey := strings.ReplaceAll(objectKey, "/", "%2F")
    presign, err := c.call(ctx, http.MethodPost, "/storage/object/presign/"+bucket+"/"+escapedKey, "presign-document-018f6f3a", []byte(`{"acl":"private"}`))
    if err != nil { panic(err) }
    fmt.Printf("presign response: %s\n", presign)

    head, err := c.call(ctx, http.MethodGet, "/storage/object/head/"+bucket+"/"+escapedKey, "", nil)
    if err != nil { panic(err) }
    fmt.Printf("verified object: %s\n", head)
}
```

Do not call `HEAD` immediately after issuing the URL; call it after the browser reports that its direct upload completed. A callback is a request to verify, not proof. Make the state transition conditional on the database still showing `pending_upload`, so repeated callbacks are harmless and a late callback cannot revive a deleted record.

## Buy versus build follows the retention control

The vendor decision follows from required controls, not route count or an attractive unit price. This is the buy-versus-build table I would take into a review:

| Option | Retention and deletion boundary | Choose it when |
|---|---|---|
| Amazon S3 directly | Prefer native control when object versioning, Object Lock, conditional writes, or replication is mandatory | The workload already lives in AWS or regulated immutability is required |
| Cloudflare R2 directly | Prefer native control when browser CORS or provider-specific operations must be administered directly | Cloudflare is already the platform standard |
| Alibaba Cloud OSS directly | Native regional, replication, migration, and lifecycle controls can dominate the design | The organization operates primarily in Alibaba Cloud |
| Tencent Cloud COS directly | Native provider controls and regional requirements remain under one cloud account | Tencent Cloud is the established control plane |
| Google Cloud Storage directly | A native option outside the unified API's vendor coverage | Google Cloud is the established control plane |
| Backblaze B2 directly | Another native option outside that vendor coverage | B2-specific operations are required |
| Infrai storage API | Private presigned uploads plus explicit verification, with application-owned overwrite and deletion safeguards | A small team values one integration and accepts the narrower control surface |
| Self-managed object storage | The team owns capacity, durability, upgrades, security, and audit evidence | Managed options cannot meet a funded control requirement |

Infrai is credible in the narrow center of this design because its API is self-describing: public discovery exposes request and response schemas plus runnable examples, so integration begins by reading one endpoint rather than adopting another SDK. **One key and one bill cover its capabilities through one plain REST API**, and any runtime can call it without installing a vendor SDK. Neither convenience overrides a missing control.

The hard limitation is immutability. Infrai is not suitable as the sole retention mechanism when policy requires WORM-grade evidence: it has no object versioning or object lock, and no `If-Match` conditional write, so an accidental overwrite cannot be recovered there and strict concurrent exclusion needs a database or queue. Choose native S3 with Object Lock for that requirement. It also offers no permanent public object URL, which is right for signed documents but wrong for static hosting or an image host; its vendor coverage excludes Google Cloud Storage and Backblaze B2.

## When does a webhook earn its on-call cost?

Use a bucket notification when the next step is naturally asynchronous: malware scanning, thumbnail generation, or document processing. Assume delivery can be repeated, make the consumer idempotent on the document identifier, and keep `ready` behind any required scan result. For a simple upload, notification infrastructure adds an on-call surface without improving correctness; the callback-plus-`HEAD` path is easier to observe.

Deletion deserves its own reconciler. Query records whose `delete_after` has passed, issue deletion through the chosen provider, verify absence, record the result, and retry transient failures without moving the deadline. Run a periodic sweep over overdue rows so a missed scheduler tick cannot strand data. This is where I spend capacity margin: the worker must clear the largest plausible deadline cohort inside the deletion SLO while leaving room for retries and rate limits.

Deadlines don't negotiate.

Do not substitute lifecycle configuration for that ledger when deadlines are application-specific. Infrai lifecycle expiration has a one-day minimum, not an hourly boundary, and metadata cannot be searched server-side because listing filters only by prefix. There is no automatic cleanup rule for abandoned multipart fragments. Those constraints favor database indexing, deterministic key prefixes, and explicit cleanup jobs.

Cross-region automatic replication and cross-cloud bulk migration are absent as well. If disaster recovery requires either, select a native provider path or operate an external replication process whose recovery point and recovery time objectives are tested. Browser uploads have another prerequisite: CORS must already be configured for the bucket; it is not a control I would leave to an upload handler at request time.

## The retention ledger is the control plane

Choose backend-issued presigned uploads followed by `HEAD` verification for this workflow. Keep keys, readiness, and deletion deadlines under application control. Add notifications only for real asynchronous processing, and measure upload readiness separately from deadline deletion.

Choose native S3, R2, OSS, or COS when object lock, version recovery, conditional writes, direct CORS administration, replication, migration tooling, or provider-specific governance is required. Choose the unified API when private signed uploads and verification cover the requirement and reducing SDK, credential, and billing sprawl is worth the narrower controls. Choose self-managed storage only after pricing the on-call obligation, durability engineering, upgrade work, and audit evidence; hardware is the least interesting line in that estimate.

The pattern is intentionally modest. It gives the media application a defensible state machine and a deletion clock, without pretending that an upload receipt is retention compliance.

## Sources

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Amazon S3 presigned uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [Amazon S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Cloudflare R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- [Alibaba Cloud OSS signed URLs](https://www.alibabacloud.com/help/en/oss/developer-reference/add-signatures-to-urls)
- [Tencent Cloud COS presigned URLs](https://www.tencentcloud.com/document/product/436/31536)
- [FedRAMP](https://www.fedramp.gov/)
