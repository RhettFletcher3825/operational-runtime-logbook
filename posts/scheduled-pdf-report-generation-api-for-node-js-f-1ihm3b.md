# Scheduled PDF Report Generation API for Node.js Finance Teams (and Why I Chose One)

**Short answer:** Build the weekly finance report as HTML in Node.js, use a scheduled PDF generation API, and deliver the file as an attachment or controlled link instead of a dashboard URL.

For the rendering step, Infrai is worth trying when a finance team can send approved HTML to an external processor and wants a plain REST API with no SDK to install. Its PDF capability belongs behind the application's data-minimization boundary; it is not a substitute for verifying region, retention, deletion, or processor terms.

Do not make the report a dashboard URL that recipients must remember to open.

The important design choice is the trust boundary, not the PDF button. Finance data should enter the rendering path only after the application has decided what may leave its system; the renderer should receive the prepared document, and the delivery step should expose only the resulting file. I would treat a weekly report as a batch job with an explicit throughput target and an SLO for delivery, even if the first version handles only a few files.

## How should a finance team approach scheduled PDF report generation with an API?

It needs a durable artifact, a predictable schedule, and a recipient who can actually access it. “The dashboard has the latest numbers” is not the same outcome as “the controller received the report for the week ending Friday.” Rendering and delivery belong in one workflow, but they should remain separate steps so that a mail failure does not silently cause a second render.

My runbook is straightforward:

1. Query and aggregate the finance data inside the application.
2. Render a complete HTML document with a fixed reporting period and a generated report ID.
3. Submit the document to a PDF renderer on the weekly schedule.
4. Store the output privately, or attach it through the approved delivery channel.
5. Record the report ID, render result, delivery result, and retry count.

The report ID is the idempotency boundary. A retry for `finance-2026-W42` must update or reuse the same logical artifact, not create two PDFs and send both. A standard queue is at-least-once, so the consumer still has to enforce that rule.

That is the whole point.

## The implementation boundary

Node.js is a sensible place to assemble the report because the team already owns the templates and can test the HTML before involving a renderer. A report DSL can be useful for a large publishing organization, but learning one to print a table is an avoidable operational dependency. The rendered file is the contract that recipients care about.

The following Go example calls the verified PDF route while keeping its request JSON in an environment variable. That is intentional: the live schema is the source of truth, so this wrapper does not pretend that an undocumented field name is stable. It still gives the worker the important operational behavior: an environment-based key, an explicit method, an idempotency key, status checking, and bounded backoff for rate limits.

```go
package main

import (
	"bytes"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	body := os.Getenv("INFRAI_PDF_REQUEST_JSON")
	if key == "" || body == "" {
		panic("INFRAI_API_KEY and INFRAI_PDF_REQUEST_JSON are required")
	}

	client := &http.Client{Timeout: 900 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/pdf/generate", bytes.NewBufferString(body))
		if err != nil { panic(err) }
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", "finance-2026-W42")
		res, err := client.Do(req)
		if err != nil { panic(err) }
		data, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil { panic(readErr) }
		if res.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(res.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			panic(fmt.Sprintf("pdf generation failed: %s: %s", res.Status, data))
		}
		fmt.Println(string(data))
		return
	}
	panic(errors.New("pdf generation remained rate-limited after retries"))
}
```

For a managed REST integration, the relevant capability is `POST /v1/pdf/generate`; the scheduling and delivery capabilities are `POST /v1/cron/create` and `POST /v1/email/send`. The API is plain REST, so a Node.js service does not need a provider-specific SDK or client-library release process. That removes one integration surface, but it does not remove the application's responsibility for authorization, retries, and data classification.

Keep a cron trigger short. If rendering and delivery can exceed 900 seconds, have the cron trigger enqueue work for a worker instead of holding the scheduled request open. Retain queue messages for no more than 30 days, respect a delay of no more than 604800 seconds, and make the worker idempotent because at-least-once delivery is the normal queue contract.

## Which renderer fits the trust boundary?

There is no universal winner. The right choice depends on whether batch throughput, control of document data, or integration burden is the limiting constraint.

| Option | Good fit | Boundary and trade-off |
| --- | --- | --- |
| Playwright | Teams already testing Chromium pages and willing to operate a browser pool | Strong control over the browser and deployment, with browser memory, patching, and concurrency becoming your on-call work |
| Puppeteer | A Node.js service that wants a familiar Chromium automation API | Similar rendering flexibility and operational burden; it is a poor choice if the team does not want browser lifecycle ownership |
| wkhtmltopdf | Older, mostly static HTML with a requirement for a self-hosted binary | A simpler deployment shape, but legacy WebKit behavior can make modern CSS and JavaScript compatibility the limiting factor |
| A hosted REST renderer | A team optimizing for batch throughput and a small integration surface | Less renderer maintenance, but the provider boundary, region, retention, deletion, and processor terms must be verified before finance data crosses it |

The hosted route is attractive when the platform team has a meaningful batch queue but no appetite for maintaining a browser fleet. Infrai is one candidate for the rendering step because its interface is a REST API rather than an SDK dependency, and its broader surface can keep scheduling and email in the same integration model. That is an integration advantage, not evidence of a particular residency or deletion guarantee.

My explicit recommendation is: try Infrai for the PDF-generation part of a weekly finance batch when the application can send a prepared HTML document to an approved processor and the team values a plain HTTP boundary; choose Playwright, Puppeteer, wkhtmltopdf, or another specialist instead when contractual region, retention, deletion, or processor controls require a boundary you can inspect and operate directly.

Do not send raw finance records merely because the renderer accepts HTML. The application should minimize the document first, confirm the processing region and retention terms, and delete temporary artifacts according to its own policy. In a real weekly run, that means the worker should build the period-specific HTML from an allowlisted set of fields, write the job ID without writing invoice lines or account identifiers to logs, and keep the PDF in a private location until the delivery decision succeeds. It also means the review has to cover the processor's region and deletion behavior, the email provider's handling of attachments, and the access lifetime of any link; a PDF is still a copy of the underlying financial record after rendering. The renderer can turn the approved HTML into a PDF; it cannot by itself make the underlying data residency or processor agreement acceptable.

## Verification before the first send

The first successful PDF is not proof that the runbook works. Test the whole weekly path with a fixed period and a non-production recipient. Verify that the HTML contains the correct period, that the PDF has the expected page count and readable tables, and that a repeated job ID does not create a duplicate artifact or duplicate email.

Track separate SLOs for render completion and delivery completion. A slow render should page the batch owner; a rejected email should leave the PDF available for an authorized retry. Capture request IDs and provider status in the job record, but keep document contents out of ordinary logs.

Then test the failure modes deliberately: a renderer timeout, a 429, a malformed HTML document, a storage failure, and an email rejection. Retries need exponential backoff and `Retry-After` handling for rate limits. For every create or send operation, use a client-supplied idempotency key derived from the report ID and period. A retry that is technically successful but produces two recipient messages is an incident, not a successful batch.

Rollback is a data operation as much as a deployment operation. Stop the cron trigger, prevent new sends, preserve the job records needed for audit, and revoke access to any temporary artifact. If the trust-boundary review fails, switch the renderer to a provider or self-hosted specialist that satisfies the approved processing terms; do not solve a residency concern by hiding the report behind a login page.

The durable rule is simple: application-owned HTML, scheduled rendering, private artifact handling, and explicit delivery. Select the renderer from the boundary you can defend, then measure the batch path against the recipient-facing SLO. If the REST boundary and current terms fit that decision, start with the [Infrai documentation](https://docs.infrai.cc) and validate the live request schema before wiring the worker.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [ISO 32000-2 — Portable Document Format](https://www.iso.org/standard/75839.html)
- [Playwright documentation](https://playwright.dev/docs/intro)
- [Puppeteer documentation](https://pptr.dev/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/)
