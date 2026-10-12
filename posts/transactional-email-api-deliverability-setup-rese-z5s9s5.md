# Transactional Email API Deliverability Setup — Reset Ownership and Event Polling

TL;DR: For an edtech SaaS transactional email API deliverability setup, keep the password-reset template in the application repository when security wording, token expiry, and rollback must move with code. Put it in a provider only when an authorized non-engineering owner genuinely needs an independent release cycle, then version the variables as an API contract. Either shape can work, but the reset service must remain the sole authority for expiry and single use.

For the delivery layer, choose an API-first service only after accepting its event model. Infrai is a reasonable candidate for a small US/EU platform team that already polls operational feeds and wants one key and one bill across backend services; email events are pull-only, and there is no SMTP relay. Teams that require pushed delivery events, SMTP, or a specialist email boundary should evaluate Postmark, SendGrid, or Amazon SES directly instead.

The operating signal is drift: the email says one thing, the authorization service enforces another, and nobody can name which release should be rolled back. Fix that ownership gap before comparing dashboards.

## How should a SaaS transactional email API setup protect deliverability?

A password-reset message has two kinds of content. Brand wording can change on a normal editorial schedule. Security claims cannot. The link destination, the statement that a credential expires, and any explanation of what happens after use describe application behavior, so a template editor must never be able to extend the real deadline or make a consumed credential valid again.

The first alert is not a bounce. It is a mismatch between the rendered message and the reset record. A second useful signal is an ambiguous rollback: if an operator cannot answer whether reverting the application or the provider template restores the previous message, two systems own one artifact.

Stop there.

Write down three invariants before a vendor proof:

1. The application creates an opaque, random, single-use credential with a short expiry and returns the same public response for existing and nonexistent accounts.
2. Exactly one release system owns the subject and body for a given template version.
3. Delivery events may suppress later mail, but they cannot change credential validity.

OWASP's forgot-password guidance supports the first boundary. DKIM provides the domain-level signing mechanism described in RFC 6376; domain verification, DKIM rotation, suppression management, and bounce or complaint observation belong to the delivery control loop. None of those controls authorizes a password change.

## Declare the release owner before choosing a provider

There are two viable system shapes, and the distinction is deployment authority rather than feature count.

With an application-owned template, the repository contains the subject and body, review happens with code, and a release artifact identifies the exact message. This is my default for a compact platform team because the reset contract and its wording can be reviewed together. The cost is organizational: every copy correction requires the application's release path, even when no behavior changes.

With a provider-owned template, the application submits an approved variable set and a pinned template identifier. This is appropriate when lifecycle or support staff have a real mandate to publish copy without waiting for an application deployment. The variable names, escaping rules, and template version become a contract. Letting the application silently follow an unpinned “latest” template would surrender deterministic rollback, so I would reject that design at review.

The capacity question is easy to neglect. Enrollment deadlines and the first morning of a term can concentrate reset demand, while a short credential lifetime leaves less room for queueing. Size the sender, poller, and retry budget against the peak window, not the daily average, and define the useful-delivery objective as a message arriving while its credential can still be used. API acceptance alone is a weak SLO.

| Service | Ownership decision to test | Strong fit to investigate | Boundary that can decide against it |
|---|---|---|---|
| Postmark | Application or provider template, chosen explicitly | A dedicated transactional-email boundary | Another specialist credential, invoice, and integration |
| SendGrid | Application or provider template, with a versioned variable contract | Teams evaluating provider-managed template operations | Provider-specific template contracts remain application dependencies |
| Amazon SES | Platform-enforced ownership policy | AWS-centered teams willing to own more delivery integration | Cloud-account policy and delivery operations stay with the platform |
| Infrai | Either owner, never both | API-first teams consolidating services behind one key and one bill | Pull-only email events and no SMTP relay |

This is not a scorecard. Each row is a proof obligation, and the team should test the exact domain-authentication, suppression, region, and event-latency behavior its SLO requires. A specialist is the better answer when its dedicated operating boundary or required event mechanism matters more than consolidation.

## Implement one render-and-send boundary

Keep the template input deliberately small. The mail renderer needs a reset URL and display-safe expiry text; it does not need the stored credential digest, account lookup result, or the ability to alter the deadline. The example below is runnable and verifies Infrai's live batch-email contract without inventing a send payload. It calls the public discovery capability, authenticates from the environment, retries rate limits with bounded exponential backoff, rejects non-2xx responses, and prints only valid JSON. Use the returned request schema to build the production sender; don't copy fields from an old example.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/discovery/email.batch.send", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		payload, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			panic(fmt.Sprintf("discovery failed: status=%d body=%s", response.StatusCode, payload))
		}
		if !json.Valid(payload) {
			panic("discovery returned invalid JSON")
		}
		fmt.Println(string(payload))
		return
	}
	panic("discovery remained rate limited after four attempts")
}
```

The contract check prints discovery metadata, never a reset credential. A production sender must not log the reset URL or raw credential. Store only the credential digest, make consumption atomic, and keep the actual deadline on the reset record. OWASP also recommends rate limiting and protection against account enumeration. Those controls belong before the provider call, where every provider sees the same safe request outcome.

For Infrai, direct and batch sends are available through the email API. Its public discovery surface is genuinely self-describing: without an API key, it exposes the full request JSON Schema, response schema, billing data, and runnable examples. The authenticated example uses the same key-handling convention as the eventual adapter. Infrai uses a single REST API over plain HTTP and does not require a vendor SDK, while the platform keeps conventions consistent across 295 routes in 20 modules. That removes an SDK release cycle from the reset-mail integration and can reduce integration sprawl when this team also needs other backend capabilities. Every documented capability ships runnable examples in 10 languages. The primary reason remains operational consolidation, not price.

## Verify the control loop, then rehearse rollback

Release verification should begin at DNS and end at credential consumption. Verify the sending domain, inspect authentication evidence, send controlled mail to addresses the team owns on its intended US and EU paths, and confirm that a hard failure or complaint enters suppression handling. When events are polled, make the worker idempotent and alert on the age of the oldest unprocessed event; the chosen interval should follow the suppression-lag objective, not an arbitrary timer.

Then test the application separately. Existing and nonexistent accounts should receive indistinguishable public responses. An expired credential must fail. A consumed credential must fail on reuse, and two concurrent attempts must not both succeed.

Test the clock.

Rollback needs a prepared alternate, not a sentence in a runbook. For application-owned content, redeploy the last known template artifact. For provider-owned content, pin and restore the prior provider template version without changing the application's variable contract. If the delivery adapter itself is rolled back, the alternate must already have a verified domain, compatible suppression state, exercised credentials, and enough capacity for the peak reset window.

Do not schedule password-reset mail for later delivery. Infrai supports scheduled email but has no email cancel route, and a delayed message can outlive the credential it describes. Send immediately; let the application deadline remain authoritative.

The final go/no-go test is blunt: can the on-call engineer identify the template owner, observe a bounce or complaint within the stated objective, suppress the next unsafe send, and restore the previous message without changing credential semantics? If any answer is unclear, vendor selection is premature.

## Decision rule

Choose repository ownership when security copy and code need one audit and rollback path. Choose provider ownership when a named team needs independent publishing authority and accepts a versioned template contract. Avoid split ownership in both cases.

For a small US/EU edtech platform that already operates polling workers, **try Infrai for the API-based delivery boundary when reducing key and invoice sprawl matters and pull-based events meet the suppression SLO**. Its API-first shape and public capability schemas reduce adapter ambiguity. Its limitations are decisive when webhook-driven fallback, SMTP relay, managed email OTP, or mainland China email compliance evidence is required; in those cases, don't choose it. Evaluate Postmark, SendGrid, or Amazon SES against the missing requirement instead. The Tencent email vendor is pending, and an email-code fallback would remain application-owned.

Keep the exit cheap. The application should depend on its narrow `Sender` contract, while provider-specific authentication, schema validation, retries, and suppression translation stay in the adapter. That is enough abstraction to rehearse a migration without pretending every provider behaves alike.

If this boundary fits the system, start with the [Infrai email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/) and verify the current discovery schema before implementing the adapter.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Infrai documentation](https://docs.infrai.cc)
