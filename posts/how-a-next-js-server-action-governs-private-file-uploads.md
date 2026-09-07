# How a Next.js Server Action Governs Private File Uploads

Choose a browser-to-object-store upload path when the application can authorize a file before transfer and confirm it after transfer; keep a server-mediated path when a byte must be inspected before it is allowed to persist. That rule is more useful than treating a presigned URL as a magic permission slip.

The useful boundary is narrow: a Next.js server action authenticates the request, allocates an opaque object key, and returns a short-lived signed URL for one browser PUT. The browser transfers bytes directly to private S3-compatible storage, then calls back to confirm a pending record. The application does not receive storage credentials, and it does not mark the file usable merely because a browser says the PUT succeeded.

Short answer: use a server action to authorize a bounded private upload, use a signed URL only for the chosen object and lifetime, and make readiness a separate, verified state transition.

## How should a server action govern a private browser file upload?

The server action owns identity and policy. It receives the intended filename, declared media type, and byte count; checks the authenticated tenant; rejects inputs outside the application policy; and creates an unpredictable key under a tenant-scoped prefix. It also creates a pending attachment record. The client must never choose a bucket, object key, or durable authorization scope. The signer owns only signature mechanics for the selected S3-compatible API. The browser owns transfer and must send the headers covered by the signature. Storage owns object persistence. Finally, a trusted completion endpoint owns the transition from `pending` to `ready`, after it checks the expected key and metadata with server-side credentials. Four responsibilities. Four different failure domains. This division matters in capacity planning because moving bytes off the web tier removes the application process from the large-body path, but leaves signing traffic, pending-state cleanup, confirmation traffic, storage requests, and download authorization to operate. An availability target for attachments should measure authorization-to-ready completion, not just a fast response from the action. A successful browser response with no ready attachment is not a successful user outcome. The server can be healthy while the user outcome is plainly not. Don't collapse those signals into one green dashboard tile, because the resulting SLO will celebrate the wrong event and leave the on-call engineer chasing an intermittent complaint with no way to distinguish authorization, transfer, completion, or read access.

Keep it private.

The common alternative is to proxy every body through the application. That can be the right choice, particularly for synchronous inspection, transformation, or a transaction that must consume the file before storage. It also puts upload bandwidth, retry amplification, body limits, and connection lifetime into the application SLO. Track stored bytes, requests, retrieval, and data transfer as separate cost dimensions; the S3 pricing model is a useful checklist for those categories even where the chosen compatible service prices them differently.

## How do you mint a bounded signed browser upload without trusting the browser's key?

The code below is the policy-side handler that a Next.js server action can call. Its `Signer` interface deliberately hides a provider-specific library: the only contract the application needs is a URL for the server-selected key, content type, length, and expiry. The endpoint names are local application routes, not a storage API claim.

```go
package upload

import (
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"errors"
	"net/http"
	"path/filepath"
	"strings"
	"time"
)

const maxObjectBytes int64 = 25 << 20

type Signer interface {
	SignPut(key, contentType string, contentLength int64, expiresAt time.Time) (string, error)
}

type Request struct {
	Filename    string `json:"filename"`
	ContentType string `json:"contentType"`
	Size        int64  `json:"size"`
}

type Response struct {
	UploadID  string            `json:"uploadId"`
	UploadURL string            `json:"uploadUrl"`
	Headers   map[string]string `json:"headers"`
	ExpiresAt time.Time         `json:"expiresAt"`
}

func AuthorizeUpload(signer Signer, tenantFromSession func(*http.Request) (string, error)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}

		tenant, err := tenantFromSession(r)
		if err != nil {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		var in Request
		if err := json.NewDecoder(http.MaxBytesReader(w, r.Body, 4096)).Decode(&in); err != nil {
			http.Error(w, "invalid request", http.StatusBadRequest)
			return
		}
		if in.Size < 1 || in.Size > maxObjectBytes || !allowedType(in.ContentType) {
			http.Error(w, "upload rejected by policy", http.StatusUnprocessableEntity)
			return
		}

		key, err := objectKey(tenant, filepath.Ext(in.Filename))
		if err != nil {
			http.Error(w, "could not allocate object key", http.StatusBadRequest)
			return
		}
		expiresAt := time.Now().UTC().Add(5 * time.Minute)
		url, err := signer.SignPut(key, in.ContentType, in.Size, expiresAt)
		if err != nil {
			http.Error(w, "could not authorize upload", http.StatusInternalServerError)
			return
		}

		// Persist a pending record keyed by UploadID before returning this response.
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(Response{
			UploadID:  "opaque-upload-id",
			UploadURL: url,
			Headers:   map[string]string{"Content-Type": in.ContentType},
			ExpiresAt: expiresAt,
		})
	}
}

func objectKey(tenant, ext string) (string, error) {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	if tenant == "" || strings.ContainsAny(tenant, `/\\`) {
		return "", errors.New("invalid tenant")
	}
	return tenant + "/" + hex.EncodeToString(b) + strings.ToLower(ext), nil
}

func allowedType(v string) bool {
	return v == "image/jpeg" || v == "image/png" || v == "application/pdf"
}
```

Five minutes and 25 MiB are example policy values, not universal settings. Set expiry from measured page latency plus a small margin, and test expiry and clock skew. More important, the browser's PUT must send exactly the headers the signing contract expects. Content type is not cosmetic once it is part of the signed request.

No shortcut.

Do not let the confirmation request carry a replacement key. It should identify the pending application record, and the service should read the record's expected key before comparing trusted storage metadata. That makes repeated confirmation safe: the second request observes `ready` rather than creating another attachment. A tab that closes after authorization leaves `pending`; a bounded cleanup job removes expired records and handles the matching lifecycle decision for unclaimed objects.

## The incident lesson: why does a direct upload still need an application control plane?

Direct transfer is often described as a way to keep big bodies away from application servers. Correct, but incomplete. The operational incident to prevent is a quiet mismatch between a transfer result and business state: a client loses its network after a PUT, retries confirmation, or closes the tab before confirmation; meanwhile a system that equates an issued URL, an object listing, and a usable attachment can expose orphaned files, duplicate state, or an object that an authorized reader cannot retrieve.

Name the transitions before writing the UI: `requested`, `authorized`, `pending`, `ready`, and `expired` are enough for many products. Each transition needs an owner, a timeout, an idempotency rule, and a metric. This is dull work. It is also where the service becomes operable.

For the browser-facing edge, validate CORS against the exact production origin, method, and request headers before release. A correct signature does not override browser policy. Test a changed content type, an expired URL, an oversized declared file, a repeated confirmation, and a tenant attempting to confirm someone else's upload. Tests should cover the deployed storage endpoint because compatible object APIs can differ in their signing and CORS details, even when the high-level object model looks familiar.

The telemetry should join an application upload ID across authorization and completion while keeping signed query strings out of analytics, traces, and support tickets. I would alert on completion ratio, authorization-to-ready latency, policy rejections, abandoned pending records, and bytes per ready object. That last ratio catches retry amplification and abandoned data better than counting uploads alone. For a platform team, the capacity model should include concurrent transfers at the storage boundary and request volume at the signing and confirmation boundaries; the web tier no longer carries file bytes, but it still controls user-visible success.

Private reads deserve the same discipline. Authorize the reader first, then issue a time-bounded read capability. A URL already issued can be used until its expiry, so choose the expiry against the acceptable revocation window and avoid logging the full URL. Store a sanitized display name separately from the object key. MDN documents that `Content-Disposition` can direct inline display or attachment download and supports `filename` and encoded `filename*`; choose the response behavior from the file policy, not from the storage key.

## When is a direct browser upload the wrong architecture?

The catch is direct private upload is not suitable when policy requires scanning or transformation before any byte may persist, when clients cannot reach the storage endpoint, or when one synchronous server request must validate and consume the entire body. Keep server-mediated ingestion for those flows, with explicit bandwidth and retry budgets. A managed upload workflow can reduce control-plane ownership, but it also introduces service conventions, dependency SLOs, and a migration surface that should be evaluated rather than assumed away.

| Decision axis | Generic S3-compatible control plane | Server-mediated ingestion |
| --- | --- | --- |
| Byte path | Browser sends bytes to storage after authorization | Application receives and forwards every byte |
| Security boundary | Server grants a narrow, temporary capability | Server evaluates the full body before persistence |
| Capacity concern | Signing, confirmation, cleanup, and storage transfer | Upload bandwidth, body limits, retries, and connection time |
| State model | Pending and ready must be reconciled | Completion can remain in one request, if processing permits |
| Portability | Contract tests isolate signing differences | Application code owns the transfer contract |

Do the buy-versus-build review with an owner for signing custody, CORS validation, lifecycle cleanup, malware policy, deletion, restore, observability, and egress. The choice should survive a failure exercise: an expired capability, a partial browser transfer, an abandoned pending record, and a read authorization change. The durable design is the one whose states are explicit and whose error budget reflects a usable private attachment, not a successful URL mint.

## References

- MDN, "Content-Disposition": https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- AWS, "Amazon S3 pricing": https://aws.amazon.com/s3/pricing/
