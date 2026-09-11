# Node.js Restore Runbook: Signed URLs for Private Objects

Short answer: make a restore request a controlled operation, not a file-download feature: authorize an operator against a backup catalog, issue a brief signed URL for one immutable object, record the decision, and restore only into an isolated target before promotion.

That order matters more than the particular object-store API. A private archive can be durable for years and still be operationally useless if the team cannot identify the right recovery point, prove who retrieved it, or keep a hurried restore from overwriting live data. For an app-data backup, the useful SLO is the time to a verified, isolated copy, rather than the time until a browser begins transferring bytes.

Start with the recovery objective. Define the maximum acceptable age of data and the maximum time to make one file usable, then test the path against those limits. A signed URL is a bearer credential, so the panel must not let a caller choose an arbitrary bucket key or treat a successful `200` response as evidence that recovery succeeded.

## What should a Node.js admin panel verify before issuing a private object storage signed URL?

Treat the browser as a requester of a restore grant, not as an object-storage client with path-building privileges. The Node.js handler should accept an internal backup ID, resolve it in a server-owned catalog, and use the catalog record to determine the tenant, object key, checksum, retention status, and allowed restore destination. It then authorizes the operator for that tenant and action before asking a signer to produce a URL for the stored key. The client never supplies a raw key, bucket name, or expiry.

The control plane needs an audit event before the URL is returned. Capture the actor, backup ID, tenant or scope, decision, expiry, request ID, and intended recovery target. Those fields make an access review possible and make an incident timeline less speculative. FedRAMP is a useful reminder that authorization is a continuing operational concern, not a checkbox attached to storage configuration.

Keep the signed URL narrow: one object, a short expiry, and the HTTP method required for retrieval. Do not put credentials in application logs, browser telemetry, or support tickets. A URL that reaches a proxy log is still a credential for its remaining lifetime.

## The failure mode is usually the catalog, not the archive

Teams often call a backup job healthy after an upload completes. That measures a write, while recovery depends on a chain of separate assumptions: the catalog points to the intended version, the archive has not passed retention, the recipient can read it, the available capacity can receive it, and the contents can be applied without damaging production.

The tempting shortcut is to proxy every archive through the admin service. It works for a small export, then turns the app tier into a data plane with buffering, egress, timeout, and retry obligations it was never sized to carry. Three simultaneous multi-gigabyte requests can consume the same connection and memory budget needed by ordinary administration traffic. Capacity planning has to include that burst, or a restore incident becomes an availability incident.

Put numbers around the decision before implementation. Estimate the largest retained archive, the likely number of concurrent restore operators, the bandwidth from storage to the recipient, and the temporary disk required for both the compressed object and its validated output. Add the validation time, because a database import or archive scan is often serialized by CPU, IOPS, or application locks even when download bandwidth looks generous. Then ask which components own an SLO during that window: the admin API should continue serving authorization requests, the catalog must remain readable, the signer must be available to issue a narrowly scoped grant, and the staging target must have headroom without competing with production. A design that proxies bytes through the API adds its workers, load balancers, connection pools, and retry behavior to that dependency set. A direct retrieval design removes much of that traffic from the control plane, but it still needs a deliberately authorized endpoint and a test of range or resume behavior. This is why a restore runbook should record expected object sizes and staging capacity alongside access policy; without both, a successful small-file drill can hide the only limit that matters during a real recovery.

Test the largest retained class.

There is another quieter failure: a restore action writes directly to its final database or filesystem location because it seems to save time. It does save time until the backup is stale, belongs to the wrong tenant, or fails validation partway through. The rollback is then an improvised production repair. Don't make that the first time the procedure is designed.

| Decision | Operational benefit | Cost or boundary |
| --- | --- | --- |
| Catalog-backed, single-object grants | Limits scope and provides an audit point | Requires a maintained metadata store |
| Browser retrieves from the storage data plane | Keeps archive bytes out of the admin service | The endpoint receiving the file must be trusted |
| Isolated restore target before promotion | Makes validation and rollback concrete | Needs temporary capacity and a promotion procedure |
| App service proxies archive bytes | Centralizes transport policy | Adds data-plane SLOs, timeouts, and scaling work |

The catch is that direct retrieval is not suitable when backup contents must never reach an operator-managed endpoint. In that case, keep the recovery environment controlled and accept the additional transfer and access-control work. A signed link cannot enforce field-level redaction after it has delivered a whole encrypted archive; use a separate, authorized transformation path when the requirement is a filtered export rather than recovery.

## A small Go signer with the controls that matter

The application can be Node.js while the service that signs and verifies grants is Go. The protocol between them is deliberately boring: Node.js sends a catalog-derived request, the signer returns a short-lived URL, and a gateway verifies the same canonical fields before serving the private object. Use a real storage provider's documented signing scheme in production; this example shows the invariants without binding the runbook to a vendor.

```go
package restore

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"errors"
	"net/url"
	"strconv"
	"strings"
	"time"
)

type Grant struct {
	ObjectKey string
	ActorID   string
	ExpiresAt time.Time
}

func canonical(g Grant) string {
	return strings.Join([]string{
		"restore-v1", g.ObjectKey, g.ActorID,
		strconv.FormatInt(g.ExpiresAt.UTC().Unix(), 10),
	}, "\n")
}

func Sign(baseURL string, g Grant, secret []byte, now time.Time) (string, error) {
	if g.ObjectKey == "" || g.ActorID == "" {
		return "", errors.New("restore grant requires object key and actor")
	}
	ttl := g.ExpiresAt.Sub(now)
	if ttl <= 0 || ttl > 5*time.Minute {
		return "", errors.New("restore grant expiry must be within five minutes")
	}

	mac := hmac.New(sha256.New, secret)
	_, _ = mac.Write([]byte(canonical(g)))
	q := url.Values{}
	q.Set("actor", g.ActorID)
	q.Set("exp", strconv.FormatInt(g.ExpiresAt.UTC().Unix(), 10))
	q.Set("sig", base64.RawURLEncoding.EncodeToString(mac.Sum(nil)))
	return strings.TrimRight(baseURL, "/") + "/" + url.PathEscape(g.ObjectKey) + "?" + q.Encode(), nil
}
```

Keep the authorization decision outside this function. Cryptography can prove that a grant was issued by the signer; it cannot prove that the caller should have been issued one. The verifier should reject expired grants, compare signatures in constant time, and emit a request-correlated audit event for an allowed retrieval. RFC 9110 describes the HTTP semantics that matter once the object is served, including conditional requests and range requests; supporting them deliberately can make an interrupted download recoverable without broadening authorization.

## How do you verify a file restore and roll it back?

Verification starts before promotion. Compare the downloaded object's checksum with the checksum captured when the backup was accepted, validate that its format can be read, and load it into a scratch database, temporary volume, or disposable environment. Run application-level checks there: expected schema version, tenant identity, row counts where those are meaningful, and a startup or integrity check appropriate to the workload. A transfer status alone cannot establish any of those facts.

Then time the whole drill. Measure authorization, transfer, validation, and promotion separately against the recovery objective, because a fast download cannot compensate for an hour of manual reconciliation. Record the storage class and archive size as test inputs so a quarterly test does not quietly exercise only the smallest, easiest backup.

Rollback should be designed as a state transition. Keep the old live target intact until the isolated result passes validation, promote with an operation that can be reversed, and retain the pre-restore snapshot for the incident-review window. For a database, that commonly means restoring to a separate instance and switching traffic only after checks pass. For a file service, it can mean changing a versioned pointer rather than replacing a directory in place.

One short drill exposes more than a dashboard does.

This pattern does have a cost: it requires a catalog, a signer boundary, audit retention, and enough spare capacity to stage a recovery. Stick with a controlled recovery environment when the compliance boundary requires it; choose a direct, scoped retrieval only when the authorized recipient and endpoint are both within that boundary. The decision is about containment and a credible RTO, not a clever URL.

## Further reading

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [FedRAMP: Federal Risk and Authorization Management Program](https://www.fedramp.gov/)
