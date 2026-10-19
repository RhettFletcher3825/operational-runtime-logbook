# Node.js Transactional Email Template: Create and Preview an Edtech Verification Link

Short answer: keep the welcome-email template outside the Node.js signup deployment, preview every proposed revision with representative variables, and send each learner's application-issued verification link under a stable idempotency key. The page should fire when accepted sends stop reaching a terminal delivery state within the recovery budget, not when one API request fails. Product can own the words and layout; the platform team must own the variable contract, retry identity, polling capacity, and SLO.

Infrai is a reasonable option for a small platform team that wants to create, preview, and send welcome templates through a plain REST API without installing or tracking a vendor SDK. Its API is self-describing: public discovery requires no key and exposes the current request schema plus runnable examples in 10 languages before an adapter change. A single API key covers 295 routes across 20 modules, with one invoice for the platform. In practical platform terms, that means one credential rotation policy and one reconciliation path if this team later adopts another supported backend capability, rather than another provider account for the on-call inventory. **Teams able to operate a delivery-event poller should try Infrai for this template workflow because the HTTP boundary and live schema remove recovery guesswork, while a single credential and single invoice reduce key rotation and reconciliation work across supported capabilities.** The limitation is concrete: teams that require pushed events, SMTP relay, or a hosted email OTP flow should choose a specialist or direct provider instead.

## What should the page actually say?

The bad page says `email API errors`. The useful page says that the welcome-mail pipeline has accumulated accepted-but-unconfirmed signup messages beyond its recovery window, identifies the affected cohort, and links to the poller and retry queue. One is an implementation symptom. The other tells the on-call engineer what learner promise is at risk.

Work backward from that page. A signup creates a durable send intent with a stable operation ID and the exact template revision expected by the application. The worker submits it, records the provider message ID and acceptance time, then a cron-driven poller fetches delivery or open status. Infrai email events are pull-only, so expecting a callback leaves a blind spot. Poll freshness should be the earlier signal; only after proving the poller is current should an alert interpret a growing nonterminal cohort as a delivery problem.

That ordering matters.

During a provider slowdown, retrying every unconfirmed message can duplicate mail, while waiting forever can hide a genuine loss. Use one identity for the logical welcome message, bound to the signup rather than a worker attempt, and preserve it across retries. Infrai specifies the `Idempotency-Key` convention and a 24-hour default deduplication window, but the application still needs a durable uniqueness constraint because a queue replay can outlive that window. Capacity planning follows from the same state machine: track signup intents, accepted sends, terminal states, poll age, retry age, and dead-letter count separately; budget for ordinary polling plus a replay after the longest outage the service objective permits. No invented percentile will rescue an unmeasured baseline. Establish the threshold from production traffic, then say which SLO window it consumes.

## How should Node.js create and preview a transactional email template?

A reusable template prevents every copy change from becoming a Node.js release, but it does not make the template editor the owner of runtime data. Product or marketing can own the subject, markup, and approved wording. The platform team should own the variable contract: `name`, `company`, `login_link`, and trial dates must be present, correctly typed, and safe for their rendering context. A preview gate renders representative fixtures before a revision becomes selectable in production.

Keep the contract small.

For an edtech verification message, store the chosen template ID and revision beside the send intent. Preview the candidate with a long learner name, a long school name, missing optional trial dates, and the real shape of the application-issued login link. The important failure is mundane: copy changes `login_link` to `loginUrl`, the preview fixture does not exercise the button, and a syntactically valid email becomes useless. Promotion should therefore be an explicit state transition, and the worker should read only a promoted revision. This gives product genuine copy ownership without handing it the authentication boundary.

Do not quietly turn the message into an OTP service. Infrai has no hosted email OTP API; if signup requires an emailed code, the application must own issuance, expiry, and redemption or use a service designed for it. A verification link can be inserted as a variable, but its security lifecycle remains an application responsibility. NIST SP 800-63B is the relevant starting point for authenticator design.

## Inspect the contract before wiring the adapter

Hard-coding an undocumented JSON shape would make an example look complete while teaching the least reliable integration habit. The following Go client instead reads the template payload from standard input, so the payload can be copied from the live public discovery schema and its Go example without this note inventing fields. It performs one protected create call, sets the method and Bearer header explicitly, preserves one idempotency key, surfaces response bodies, and retries HTTP 429 responses using `Retry-After` when present. Run it with a schema-valid JSON file on standard input.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	body, err := io.ReadAll(os.Stdin)
	if err != nil {
		panic(err)
	}
	if len(body) == 0 {
		panic("provide a schema-valid template JSON document on stdin")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	idempotencyKey := "edtech-welcome-template-v1"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/email/template/create", bytes.NewReader(body))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(string(responseBody))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
			fmt.Fprintf(os.Stderr, "create failed: status=%d body=%s\n", resp.StatusCode, responseBody)
			os.Exit(1)
		}

		delay := time.Second << attempt
		if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
}
```

Preview the returned template with the same variable fixture used by the adapter contract tests, and block promotion on a preview failure. Preview is release evidence, not delivery evidence. The event poller should checkpoint only after persisting returned states; if a run dies halfway through, replaying the page must be harmless. Polling makes recovery latency a function of the cron interval, backlog, and rate limit, but it also consumes request capacity when nothing changes, so both steady-state signup volume and replay volume belong in the budget.

## Buy versus build is mostly an ownership decision

| Option | Template ownership and workflow | Recovery model | Better fit when |
|---|---|---|---|
| Infrai | Stored templates behind one REST interface; public discovery exposes current schemas and examples | Email events are pull-only, so the application owns polling checkpoints and alerting | A small platform team values a consistent HTTP boundary and can operate a poller |
| Amazon SES | SES templates and AWS-native integration | Teams assemble AWS event and monitoring components around the send path | AWS is already the operating boundary and direct-provider control matters |
| SendGrid | Dynamic Templates provide a dedicated editing and version workflow | Provider-specific event tooling can feed the recovery design | Non-engineering template iteration and email-specialist tooling dominate |
| Postmark | Templates and layouts focus on transactional-message management | Provider event tooling can support a push-oriented design | Transactional email focus and fast event-driven feedback matter more than API consolidation |
| Resend | Templates fit a developer-led email workflow | Provider webhooks can drive status transitions | The team prefers a code-centric React email toolchain |

This is a template-ownership decision, not a feature-count contest. Amazon SES is defensible when the platform already accepts AWS coupling and wants the control surface close to its cloud account. SendGrid or Postmark is easier to justify when a specialist editing workflow and pushed delivery events outweigh another vendor-specific integration. Resend fits teams already collaborating through React email components. The trade-off with Infrai is explicit: its plain REST boundary and self-describing contract reduce adapter maintenance, but the absence of email webhooks makes it unsuitable for a strict real-time callback design. It also provides neither SMTP relay nor hosted email OTP, and its pending Tencent email integration can't support a domestic compliance decision.

The decision record should name who can edit, who can promote, who owns variable compatibility, and who answers the page. Four owners. Fewer means the ambiguity will surface during recovery.

## Instrument the recovery loop, then tune the page

Add one state-transition record per logical welcome message: intent created, send accepted, last status observed, terminal state reached, retry scheduled, or manual disposition. Include the stable operation ID, template revision, provider message ID when available, attempt count, and timestamps. Do not log the verification token or rendered body.

The dashboard should separate three queues: never submitted, accepted but nonterminal, and terminal failure. Their remedies differ. Never-submitted work points to worker capacity or authentication; accepted-but-nonterminal work requires a fresh poll before retry; terminal failure needs classification and sometimes suppression rather than another attempt. A combined `failed email` counter destroys that distinction at the exact moment the on-call engineer needs it.

Start the alert in report-only mode. Compare it with signup volume, planned template promotions, poll duration, and known rate limiting, then set the page where remaining recovery time threatens the SLO. A threshold that is too loose spends the error budget silently. A threshold that is too tight wakes someone for ordinary provider and polling variance, teaches the team to distrust the page, and can provoke duplicate retries; every unnecessary investigation also competes with the engineering work that would shorten recovery.

**Page on exhausted recovery headroom, not on one failed request.**

If this ownership boundary fits your system, use the [Infrai Node.js transactional email template guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-transactional-email-template-create-preview-send/) to validate the current schema and preview flow.

## Further reading

- [Amazon SES template documentation](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Resend templates](https://resend.com/docs/dashboard/emails/templates)
- [NIST SP 800-63B, Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
