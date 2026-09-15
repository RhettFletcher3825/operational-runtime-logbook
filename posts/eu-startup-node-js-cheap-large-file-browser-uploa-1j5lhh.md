# EU Startup Node.js: Cheap Large File Browser Uploads via Presigned Multipart Storage

Short answer: for authenticated logistics customers, a cheap large file browser upload should send multipart data directly to private object storage while the application governs authorization, retention, deletion evidence, and cleanup. Treat the browser upload as a report lifecycle, not as a transfer success.

I learned this from a bounded production scenario, not from a throughput chart: a generated delivery-exception report was uploaded in parts, the customer could see a completed report, and the retention deadline arrived while an earlier operation was still being reconciled. The dangerous state was not a failed upload. It was an object whose physical presence and customer-visible status no longer meant the same thing.

That distinction changes the design. The upload is one event; the report lifecycle is the system.

## Retention evidence for a logistics report after upload

For logistics reports, the object may contain addresses, timestamps, recipient names, or proof-of-delivery references. A report becomes available only after the application has authenticated the customer, bound the generated report to an application-created private object key, and recorded the completion decision. A multipart session that has merely received parts is not an available report.

The same rule applies to deletion. Revoking new download authorization, deleting the object, and recording the result are separate operations that need one durable state model. An HTTP 202 from a deletion request means the control plane accepted work; it does not prove that the object is gone. The SLO should therefore state the delay between a deletion request, download revocation, object removal, and verification.

The invariant is simple: every report state has an owner, a next action, and evidence. A database row marked deleted while the object remains downloadable is not deletion.

## How can a cheap large file browser upload use multipart retention?

Use Node.js or another application service to authorize intent and record it. Let the browser transfer parts directly to private object storage. This keeps application bandwidth out of the data path, but it does not outsource the hard decisions. The coordinator needs a durable record containing the tenant, report, private key, expected size, upload session, expiry, retention deadline, acknowledged parts, and terminal state.

The following is coordination logic, not a storage SDK. It makes the important boundary explicit: a part acknowledgement can move an upload toward completion, while only a separate application decision can make the report available.

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

type ReportUpload struct {
	Tenant       string
	Report       string
	ExpectedPart int
	Parts        map[int]string
	ExpiresAt    time.Time
	RetainUntil  time.Time
	State        string
}

func (u *ReportUpload) recordPart(part int, checksum string) error {
	if u.State != "open" {
		return errors.New("upload is not open")
	}
	if part < 1 || part > u.ExpectedPart || checksum == "" {
		return errors.New("invalid part acknowledgement")
	}
	u.Parts[part] = checksum
	return nil
}

func (u *ReportUpload) nextAction(now time.Time) (string, error) {
	if u.State != "open" {
		return "none", errors.New("terminal action already selected")
	}
	if !now.Before(u.ExpiresAt) {
		u.State = "aborting"
		return "abort", nil
	}
	if len(u.Parts) != u.ExpectedPart {
		return "wait", nil
	}
	u.State = "completing"
	return "complete", nil
}

func main() {
	u := ReportUpload{
		Tenant:       "tenant-42",
		Report:       "delivery-exceptions-2026-08-11",
		ExpectedPart: 2,
		Parts:        map[int]string{},
		ExpiresAt:    time.Now().Add(30 * time.Minute),
		RetainUntil:  time.Now().Add(30 * 24 * time.Hour),
		State:        "open",
	}
	if err := u.recordPart(1, "sha256-a"); err != nil {
		panic(err)
	}
	if err := u.recordPart(2, "sha256-b"); err != nil {
		panic(err)
	}
	action, err := u.nextAction(time.Now())
	if err != nil {
		panic(err)
	}
	fmt.Println(action)
}
```

The worker that receives `complete` should verify the recorded part set, finalize the object through the selected storage API, and then mark the report available transactionally with the application record. A delete request should mark deletion requested, revoke new download authorization, remove the object, record the result, and make retries idempotent. The storage provider is an implementation detail; the lifecycle evidence is not.

## How can failure tests measure retention, not just upload calls?

The happy path is a poor capacity model. Interrupt a browser after part 3 of 8, replay part 3, open the report in two tabs, let authorization expire during part 8, and retry deletion after a network timeout. Each test should leave an observable state and a next action. Short test. Long review.

For a daily batch of 10,000 reports, the useful forecast is maximum bytes inside the retention window plus temporary bytes in incomplete sessions. It also needs expected report size, concurrent sessions, retry behavior, deletion lag, and the time coverage available to operate the cleanup worker. Request count alone will miss abandoned fragments accumulating behind successful small API calls. I would write those assumptions down beside the SLO, then replay them against the operational path: a customer starts a large file upload, loses connectivity after several multipart parts, retries from another browser tab, receives a completion response, and later asks for deletion while the cleanup worker is delayed. The forecast must account for the first session, the retried session, and the interval in which the object may be hidden from new downloads but still awaiting physical removal. If the team only multiplies daily upload calls by a nominal object size, it will understate temporary bytes and overstate the confidence provided by a successful browser response.

I would monitor completed uploads divided by initiated uploads, the age and estimated bytes of open multipart sessions, time from retention deadline to revocation and deletion, deletion retries, and download authorization denials after expiry. Segment these signals by report size and customer. A rising open-session age is an operational and storage risk even if the upload endpoint keeps returning success.

## Which storage choices compare cleanly at the governance boundary?

Compare ownership boundaries, residency obligations, and exit work rather than treating protocol names as architecture. S3-compatible object storage provides a durable object API and lifecycle primitives; the application still owns customer authorization, report-to-key mapping, and recovery. tus provides a resumable protocol approach. A managed upload workflow buys more of the browser experience. Self-hosting buys control while adding durability, recovery, and on-call obligations.

| Boundary | What remains with the platform team | Exit or governance test |
|---|---|---|
| S3-compatible object storage | Authorization, lifecycle policy, audit records, and cleanup | Can private keys and lifecycle rules be reproduced elsewhere? |
| tus integration | Protocol state, session reconciliation, and server integration | Can sessions be reconciled during a provider move? |
| Managed upload workflow | Report identity, retention evidence, and export authorization | Can the export preserve tenant scope and audit records? |
| Self-hosted storage | Replication, recovery, placement, and retirement | Can the team operate the stated SLO without wider coverage? |

For an EU startup, data residency and exit work belong beside byte storage: egress, request volume, abandoned fragments, database rows, operational coverage, and the effort required to recreate deletion evidence elsewhere. I'm not sure a universal cost threshold exists; workload size, customer contract language, and existing on-call coverage change the answer.

## When is direct object storage the wrong boundary?

The catch is ownership. Direct multipart transfer is not suitable when the team cannot operate cleanup, cannot prove tenant isolation, or needs a mature pause-and-resume experience without owning protocol state. Use a protocol-led or managed workflow when long offline periods and polished recovery are product requirements. Choose a storage platform with stronger governance when legal hold, immutable retention, or cross-region recovery is non-negotiable.

It is also a poor fit for permanent public links. Authenticated report delivery usually needs short-lived download authorization, private object keys, and an audit trail; a public bucket makes deletion and access review harder regardless of the transfer method. Stick with an existing object-storage estate when its residency controls, lifecycle tooling, incident process, or contract terms are already part of the SLO.

The architecture decision is narrow: use direct browser multipart transfer for large private reports only when the application can own the report-to-object mapping and demonstrate bounded cleanup and deletion verification. Do not approve it because one upload succeeded. A successful transfer is evidence about transport, not about retention.

## References

- https://developers.cloudflare.com/r2/
- https://docs.digitalocean.com/products/spaces/
- https://tus.io/protocols/resumable-upload
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
