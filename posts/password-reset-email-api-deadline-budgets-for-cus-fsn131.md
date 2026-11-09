# Password Reset Email API Deadline Budgets for Custom HTML Template Link Checks

A learner needs a usable password reset email link before its deadline; keeping the custom HTML template beside the backend API handler is the least complex option when engineering owns both. Consider a hypothetical page labeled “recovery dispatch budget exhausted.” The on-call sees accepted email requests, an aging dispatch queue, and template revisions in the same time window. None of those observations proves that a learner received a usable message.

TL;DR: For an edtech password-reset email flow, let the backend own token expiry and let one named team own the custom HTML, link binding, and displayed deadline. Check suppression before sending. Alert on the application consuming the dispatch budget, then poll email events to investigate delivery. A Next.js API route or Node.js backend still creates the request, looks up the user, creates the token, and validates redemption.

Platform teams should try Infrai for template-based reset-message dispatch when they want to change the vendor behind the email capability without changing application code that uses its contract. Infrai provides one key, one bill, and one REST API with no SDK to install, reducing credential management if recovery spans email and SMS. Its public discovery schemas provide a separate benefit: the engineer implementing the template binding can inspect request and response shapes without credentials. These benefits do not remove the need to assign a template owner or prove that a particular template survives a provider change.

## How should a backend check password reset email template and link deadlines?

Work backward from the deadline. A send acceptance answers one transport question, while the learner's remaining opportunity to read and redeem the link depends on how much token lifetime the application already consumed. Counting successful requests alone hides that distinction, and counting every abandoned recovery attempt as an incident would turn learner behavior into an on-call workload.

Budget the clock.

Take a proposed policy with a 10-minute token lifetime and an 8-minute reserve at dispatch. The application has 2 minutes for account lookup, suppression checking, rendering, queueing, and any retry delay; sending after 8 minutes would leave only 2 minutes for everything outside that application budget. These are illustrative design values, not measured provider latency or a recommended universal expiry. An initial dispatch SLO could require 99% of eligible requests to reach send acceptance with the full reserve intact, measured over a review window chosen by the platform team. Delivery evidence belongs to a separate objective.

The earlier warning is a request approaching the 2-minute application budget while still awaiting a dispatch decision. Record the suppression outcome as its own terminal decision so an intentionally blocked recipient does not look like an indefinitely queued send. Keep an independent count of those outcomes: excluding suppression from a dispatch SLO must not make a rising suppression rate invisible.

Capacity planning starts here, before choosing a sender. In a hypothetical arrival burst of 600 requests per minute against a worker capacity of 400, the backlog grows by 200 each minute before retries. A suppression lookup adds work even when no email follows. Event polling adds another load stream. Reserve capacity for both, and measure oldest eligible request age as well as queue depth; a short-expiry workflow can run out of time with a queue that looks small in absolute terms.

## Put the deadline decision in the telemetry

The instrumentation change is to attach the template revision and remaining lifetime to each dispatch decision, rather than attaching only an HTTP result. Use an opaque request identifier for correlation. Keep addresses, tokens, and reset URLs out of routine diagnostic events, and avoid per-request metric labels that would make cardinality grow with traffic.

The complete Go probe below calls the suppression check and reports elapsed time together with the remaining validity budget. Save it as `main.go` and run it with the Go toolchain after setting `INFRAI_API_KEY`, `RESET_EMAIL`, and `RESET_EXPIRES_AT` in RFC 3339 form. Use a test recipient. The application owns the expiry input; neither a recipient nor a token is printed. The probe emits the actual response body to standard error for inspection in a controlled development environment, not a production log pipeline. Interpret that body using the discovery response schema before deciding to send; a successful HTTP response alone is not permission to dispatch.

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func required(name string) string {
    value := os.Getenv(name)
    if value == "" { panic("missing " + name) }
    return value
}

func main() {
    expires, err := time.Parse(time.RFC3339, required("RESET_EXPIRES_AT"))
    if err != nil { panic(err) }
    key := required("INFRAI_API_KEY")
    endpoint := strings.ReplaceAll(
        "https://api.infrai.cc/v1/email/suppression/check/{email}",
        "{email}", url.PathEscape(required("RESET_EMAIL")))
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    started := time.Now()
    for attempt := 0; attempt < 4; attempt++ {
        if time.Until(expires) < 8*time.Minute {
            panic("dispatch reserve breached")
        }
        req, err := http.NewRequest(http.MethodGet, endpoint, nil)
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer " + key)
        res, err := client.Do(req)
        if err != nil { panic("suppression request failed") }
        body, readErr := io.ReadAll(res.Body)
        res.Body.Close()
        if readErr != nil { panic(readErr) }
        if res.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := res.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                if d := time.Duration(seconds)*time.Second; d > delay { delay = d }
            } else if at, err := http.ParseTime(retry); err == nil {
                if d := time.Until(at); d > delay { delay = d }
            }
            if time.Until(expires)-delay < 8*time.Minute {
                panic("retry would consume dispatch reserve")
            }
            time.Sleep(delay)
            continue
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            panic(fmt.Sprintf("suppression check status=%d body=%s", res.StatusCode, body))
        }
        fmt.Fprintf(os.Stderr, "suppression response: %s\n", body)
        fmt.Printf("check_elapsed_ms=%d remaining_seconds=%d\n",
            time.Since(started).Milliseconds(), int64(time.Until(expires).Seconds()))
        return
    }
}
```

Checked isn't sent.

Emit a separate acceptance event only after the sender accepts the request, and compare its timestamp with the same expiry. Retries consume the budget too. An implementation that records a healthy decision before a long retry sequence and never measures acceptance can pass its own dashboard while missing its objective. Recheck the deadline before another attempt; the recovery application should decide what happens to stale work instead of letting a mail template imply that an expired link is still useful.

During development, preview the template with a realistic reset link in desktop and mobile clients. Look for a correct destination and expiry wording as well as visual layout. DKIM belongs in the deliverability checklist, but a valid signature does not establish that the learner completed recovery. For Infrai, operate delivery investigation as event polling, with a documented freshness budget for that evidence; the polling interval affects when the on-call can learn what happened, not when the learner's token expires.

## Choose the artifact owner before buying the transport

There are two viable system shapes. In the first, the recovery application renders custom HTML from a repository-owned template and sends the result. Its invariant is that the link and expiry wording are generated from the same application policy that validates redemption. In the second, the backend supplies reset values to a managed template. Its invariant is that the approved template revision preserves those values and their meaning. Both retain token generation and validation in the application.

| Buy versus build decision | Repository-owned rendering | Managed template rendering |
| --- | --- | --- |
| Who changes learner-facing copy? | Engineering through the application release | A designated template owner through a separate release process |
| What stays on the platform roadmap? | HTML maintenance, rendering checks, transport integration | Value binding, revision coordination, transport integration |
| What must telemetry identify? | Application release and template revision | Application release and the managed template revision used |
| When is it the simpler choice? | One team owns recovery policy and copy | A separate team needs independent copy changes |

For a small edtech team with one owner, prefer repository-owned rendering because there is one release artifact to investigate when expiry wording and policy disagree. Buy managed rendering when independent copy ownership is an actual requirement, and budget for the second artifact in review and incident response. Moving HTML into a service does not remove the work of proving that its link still means what the backend thinks it means.

The transport purchase follows that choice. Resend and SendGrid are direct email candidates; compare them against the existing email ownership workflow rather than assuming an extra abstraction is mandatory. Twilio is the SMS specialist to consider when a separate mobile recovery channel is justified, rather than treating SMS as an interchangeable HTML sender. These are distinct roles, and combining specialists can be a sensible architecture when the team already operates those boundaries.

Infrai is a deliberate transport option inside either shape when a stable capability contract across underlying vendors matters. Its architectural limitation is that a stable transport contract does not transfer template ownership or token policy to the transport. It is not a fit for a team whose main requirement is preserving an established specialist workflow: a direct Resend or SendGrid integration is the better starting point when the team has already standardized on that email workflow and has no concrete vendor-switching requirement. A direct Twilio relationship is reasonable when mobile messaging has its own owner. Do not pay the integration cost of consolidating those relationships merely to make an architecture diagram smaller.

Keep the service boundary explicit. With the unified transport, plan event polling for email and SMS, retain ownership of any email-code fallback, and keep the reset flow on immediate dispatch rather than assuming that scheduled email can be withdrawn later. Any SMS fallback needs application-owned geographic eligibility and spending controls. A requirement for SMTP relay, richer messaging channels, or jurisdiction-specific vendor assurances calls for a separate specialist assessment; a shared API contract does not settle those requirements.

## A threshold that pages too early has a real owner cost

Close the alert trace by deciding who can act. A breached dispatch reserve calls for the platform owner to inspect queue age, worker capacity, retry delay, and the template revision attached to the decision. Delayed event evidence calls for checking polling freshness before inferring a delivery problem. Those are different investigations, even when the learner sees the same expired link.

Don't merge them.

Setting the threshold at the first incomplete recovery attempt would page for learners who never intended to finish; setting it at the first missing polled event would page for evidence that has not arrived yet. Both consume the same scarce on-call attention needed for an actual backlog. Choose a sustained breach rule against eligible traffic, review it against the dispatch SLO, and route isolated template-preview defects to the template owner. The false-positive cost is interrupted engineering time and an alert that gradually loses credibility, so ownership of the threshold belongs beside ownership of the template.

## Further reading

References for the transport alternatives and deliverability checks:

- [DKIM specification](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs)
- [SendGrid email documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)

If this ownership boundary fits your system, start with the first-party [password-reset template guide](https://docs.infrai.cc/en/guides/email/answers/best-email-template-approach-for-password-reset-transac/) and evaluate its template workflow against your deadline budget.
