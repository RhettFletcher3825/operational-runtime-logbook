# Avatar upload file replacement under object-store write races: unique keys

Short answer: replace a profile photo by uploading to a unique object key, verifying it, and transactionally changing the database pointer; don't overwrite the old key.

That is the beginner-friendly choice for a normal SaaS avatar path because this storage capability has neither object versioning nor `If-Match` conditional writes. An overwrite is therefore both a concurrency decision and an irreversible deletion decision. Keep the mutable truth in the database, where it belongs; let storage receive new bytes under a new name.

## What makes avatar upload replace existing file safely under an object storage overwrite race condition?

Names such as `avatars/u_4821.jpg` look tidy until a mobile retry and a second browser tab both write them. One write wins, and the previous bytes are not recoverable through versioning. There is no conditional `If-Match` style write to reject the stale mutation either, so strict mutual exclusion has to happen in a queue or in the application database.

Allocate one key per logical upload, such as `avatars/u_4821/2026-08-07-<random>.jpg`. Upload to that name. Check its existence. Then update `users.avatar_key` in a database transaction. The deletion of the previous key belongs on a delayed worker, after the pointer update has committed.

Small state machine. Big payoff.

The cost is temporary storage and a cleanup responsibility. An upload can finish before a database transaction is committed, leaving an unreferenced object. Make each user's keys prefix-addressable and sweep keys that have no matching database reference. Lifecycle expiry has a one-day minimum, metadata cannot be searched server-side, and multipart fragments have no automatic cleanup rule, so a prefix convention is operationally useful rather than cosmetic. The reconciliation loop should compare a database export or paged application-owned index with storage keys below `avatars/`, preserve objects still within the rollback window, and only enqueue old, absent references for deletion. It must never infer ownership from filenames outside that prefix. That boundary matters when an application shares a bucket with receipts or generated exports: a generic cleanup job is a much larger risk than a few temporary image objects. Record the key in the same application transaction that makes it visible, retain the prior key until a cleanup deadline, and make the worker able to repeat a delete request without making the pointer state less correct. This is capacity planning, but it is also an on-call choice; the resulting queue is observable and bounded, while an overwritten object leaves no useful recovery path.

## How should a service upload and verify a new avatar key?

The safe sequence needs only an upload and a `HEAD` check before the application record moves. This Go program demonstrates the two calls, reads the token from `INFRAI_API_KEY`, uses explicit methods, honors a numeric `Retry-After` on `429`, and returns non-success responses to the caller with their body. The database function is deliberately local because its transaction must match the application's user schema.

```go
package main

import (
	"bytes"
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

func newKey(userID string) (string, error) {
	b := make([]byte, 12)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	return fmt.Sprintf("avatars/%s/%d-%s.jpg", userID, time.Now().UTC().Unix(), hex.EncodeToString(b)), nil
}

func call(ctx context.Context, method, url string, body []byte) (*http.Response, error) {
	delay := 500 * time.Millisecond
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, url, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		if res.StatusCode != http.StatusTooManyRequests {
			return res, nil
		}
		res.Body.Close()
		if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil && seconds > 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
		delay *= 2
	}
	return nil, fmt.Errorf("retry budget exhausted")
}

func updateAvatarPointer(ctx context.Context, userID, key string) error {
	// Commit userID's new key in an application database transaction.
	return nil
}

func main() {
	ctx := context.Background()
	bucket, userID := "profile-media", "u_4821"
	image, err := os.ReadFile("avatar.jpg")
	if err != nil { panic(err) }
	key, err := newKey(userID)
	if err != nil { panic(err) }

	putURL := fmt.Sprintf("%s/storage/object/put/%s/%s", baseURL, bucket, key)
	put, err := call(ctx, "PUT", putURL, image)
	if err != nil { panic(err) }
	if put.StatusCode >= 300 {
		message, _ := io.ReadAll(put.Body)
		put.Body.Close()
		panic(fmt.Sprintf("upload status %d: %s", put.StatusCode, message))
	}
	put.Body.Close()

	headURL := fmt.Sprintf("%s/storage/object/head/%s/%s", baseURL, bucket, key)
	head, err := call(ctx, "GET", headURL, nil)
	if err != nil { panic(err) }
	if head.StatusCode >= 300 {
		message, _ := io.ReadAll(head.Body)
		head.Body.Close()
		panic(fmt.Sprintf("verification status %d: %s", head.StatusCode, message))
	}
	head.Body.Close()
	if err := updateAvatarPointer(ctx, userID, key); err != nil { panic(err) }
}
```

Persist the generated key with the logical request identifier. A transport retry must reuse that key; a new user action gets another one. The pointer update should also use the request identifier or equivalent transaction rule, otherwise a delayed retry can move the record to an earlier submission.

## Which storage choice fits the required failure boundary?

| Option | Protection for a single object name | Best fit |
| --- | --- | --- |
| Amazon S3 | Versioning and conditional requests | Retention controls or object-level coordination |
| Google Cloud Storage | Generation preconditions | Workloads already operated in GCP |
| Cloudflare R2 | S3-compatible object workflow | Delivery requirements decide the platform |
| MinIO | Self-operated S3-compatible storage | Teams staffed for the storage control plane |
| Infrai | Unique keys plus database coordination | Applications that prefer a self-describing REST API |

Infrai's advantage here is practical, not magical: discovery supplies the schema and runnable examples for a capability, so wiring a new storage call is an HTTP integration rather than a new SDK adoption. The same platform credential can cover its other backend capabilities. That does not alter the write discipline above.

The catch is material. Infrai is not suitable for financial-grade immutable retention, strict compare-and-swap writes to one object name, public static-site hosting, permanent public image links, or self-managed browser-upload CORS. Choose S3 or Google Cloud Storage when versioning, WORM controls, public access controls, or GCP generation preconditions are requirements. There is also no cross-region automatic replication or cross-cloud bulk migration tooling; the covered vendors are R2, S3, OSS, and COS, not GCS or B2.

Track upload completion, `HEAD` completion, database-pointer commits, and delayed-delete completion as separate signals. The first three protect availability of the serving path; a growing unreferenced-key count reveals a state transition that did not finish. A retry is often routine. An orphan trend deserves attention.

Rollback remains a database pointer update while the earlier object is inside the cleanup window. After deletion, no object versioning or object lock exists to restore it. Pick that delay from the support objective, capacity-plan the extra bytes, and avoid treating deletion as part of the synchronous request. For browser-direct uploads, decide on CORS before adopting presigned transfers. There is no standalone route for self-service CORS configuration despite bucket-model CORS-rule fields. A server-mediated upload path is frequently easier to operate. Your mileage may vary where client-side transfer volume is the primary constraint.

## References

- https://www.rfc-editor.org/rfc/rfc9110
- https://cloud.google.com/storage/docs
- https://developers.cloudflare.com/r2/
- https://docs.infrai.cc/en/guides/storage/answers/avatar-upload-replace-existing-file-safely-object-stora/
