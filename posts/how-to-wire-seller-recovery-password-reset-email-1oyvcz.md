# How to Wire Seller Recovery (Password Reset Email API Without SMTP Relay)

A new-order notification is useful only if the gaming marketplace seller can get back into the account. A beginner Node.js application's password reset email API versus SMTP relay decision should therefore start with the transport its authentication component actually accepts.

Short answer: choose an HTTP email API when the backend owns the reset flow and can call a send operation directly; choose an SMTP-capable provider when the auth component requires SMTP. An adapter is extra integration work, and the seller waiting to fulfill an order does not benefit from that work unless it buys something concrete.

Consider a bounded production review scenario, rather than a claimed incident: a seller receives an order alert, cannot sign in, requests recovery, and receives another order while the first request is still being processed. The lesson to extract is that order notifications and account recovery need separate message identities, even when they use the same recipient and transport. An accepted send request also needs to remain distinguishable from a delivery event.

Keep those boundaries explicit.

Transport isn't authorization.

## Should a beginner app use an email API or SMTP relay for password reset?

Start with the existing password reset callback. If it hands your application a recipient and a reset message to deliver, an HTTP integration can be a small piece of backend code. If it accepts only SMTP credentials, using an HTTP-only provider means owning a transport adapter as well as the recovery workflow. For a beginner application, that maintenance obligation belongs in the selection decision before templates or channel breadth.

The gaming marketplace adds a second seam: the new-order notification identifies the seller account, while recovery must use the account's verified recovery destination. Do not promote the address on an incoming order payload into an authentication authority. Define that account lookup in the backend, then let the existing authentication system own the reset credential and its validation. An email delivery API does not establish that a password change is authorized.

Here is a proposed acceptance test, not an observed benchmark: submit the same recovery job twice for seller seller-42 after order order-731, then submit a separate order alert. The two recovery attempts should share one durable operation identity; the order alert should have its own. Measure an application SLO from recovery request to send acceptance separately from the time until a delivery event becomes visible. Set both targets explicitly during rollout.

No latency claim follows from choosing a transport.

Acceptance isn't delivery.

## 2. Compare the work you would own

Infrai uses a single API key and unified billing for account lookup, email, and SMS through one REST API, with no SDK required. Its self-describing API exposes public discovery schemas and runnable examples, so connecting a capability starts with reading its contract; suppression checks also support avoiding repeat sends to blocked or bounced recovery addresses. The trade-off is one vendor to trust, one bill, and one outage surface.

The limitation is concrete: this option does not support SMTP relay, so it is not suitable for an SMTP-only auth component without an adapter. Prefer Postmark or Amazon SES when their SMTP integration matches that component.

The buy-versus-build decision is about ownership. The alternative Clerk + Resend + Twilio stack implies three service signups and three sets of credentials; your application must connect account identifiers, email destinations, phone destinations, suppression decisions, and message outcomes across those boundaries. Counting signups is not a measurement of integration time. A team that already operates those services may prefer that glue to changing its account system.

| Option | Work to buy | Work to build or verify | When it belongs on the shortlist |
| --- | --- | --- | --- |
| One discovery-based REST platform | Shared credentials and discoverable contracts across the account, email, and SMS seam | Recovery policy, destination mapping, polling, and cross-channel suppression policy | A custom backend can call HTTP directly and the shared trust boundary is acceptable |
| Clerk + Resend + Twilio | Separate account, email, and SMS service integrations | Three credential lifecycles plus explicit account-to-channel mapping and fallback decisions | The application already has these integrations or values independent provider choices |
| Postmark | A focused transactional email service | Verify the selected API or SMTP integration against the auth component; keep account and SMS coordination in the application | Email transport is the immediate decision and the team wants to evaluate a dedicated mail provider |
| Amazon SES | A mail service within an AWS service boundary | Verify API or SMTP configuration and account readiness; retain application ownership of recovery orchestration | Existing AWS operations make that boundary practical |
| Self-managed mail transport | Direct operational control | Delivery operations, suppression handling, monitoring, and on-call ownership in addition to application logic | A concrete requirement justifies the operational burden |

These are not interchangeable auth systems. Postmark and SES belong in the email transport comparison; Clerk and Twilio belong in the account and SMS seams. Compare the entire workflow, then buy only the parts that remove work your team would otherwise maintain.

## 3. Execute the account-to-email handoff against discovered contracts

The preventative path is to resolve the account first, take the recovery destination from that result, and pass it into the email send request with a stable operation key. Do not guess a response field: the account response envelope and the email recipient schema need to be inspected before wiring them together. The program below keeps those choices explicit through configuration and obtains methods and paths from discovery.

Save it as main.go and run it with go run main.go. It uses only the Go standard library, so it can also serve as a small contract probe before implementing the same handoff in a Node.js backend. This is an account lookup followed by message delivery; the existing auth system must already have generated the reset message.

Configure API_BASE to the provider's documented HTTPS base URL including the version prefix, INFRAI_API_KEY to its secret key, and AUTH_CAPABILITY and EMAIL_CAPABILITY to the discovery identifiers for account lookup and email send. AUTH_PATH_VALUES is a JSON object of the lookup's path substitutions, such as a user identifier. EMAIL_PARAMS is the exact send payload from the discovered schema with your sender and prepared reset message. ACCOUNT_EMAIL_POINTER is a JSON pointer into the lookup response; EMAIL_RECIPIENT_FIELD names the top-level recipient parameter in the send schema. This minimal adapter requires that parameter to accept a string; use a schema-specific adapter if it expects a structured value.

Finally, set SEND_KEY to the recovery job's durable identity. Reuse it on retries, including process restarts. A new recovery attempt gets a new identity; an order alert gets another. None of these configuration names is a claim about a provider's request fields.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

type contract struct {
    Method string
    Path string
    Idempotent bool
}

var client = &http.Client{
    Timeout: 15 * time.Second,
    CheckRedirect: func(_ *http.Request, _ []*http.Request) error {
        return http.ErrUseLastResponse
    },
}

func required(name string) string {
    value := os.Getenv(name)
    if value == "" { panic("missing configuration: " + name) }
    return value
}

func request(method, target, key, operation string, body []byte) []byte {
    parsed, err := url.Parse(target)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" {
        panic("request requires an HTTPS URL")
    }
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(method, target, bytes.NewReader(body))
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer " + key)
        if len(body) > 0 { req.Header.Set("Content-Type", "application/json") }
        if operation != "" { req.Header.Set("Idempotency-Key", operation) }
        res, err := client.Do(req)
        if err != nil { panic(err) }
        data, readErr := io.ReadAll(io.LimitReader(res.Body, 2 << 20))
        res.Body.Close()
        if readErr != nil { panic(readErr) }
        if res.StatusCode == http.StatusTooManyRequests {
            if attempt == 4 { panic("rate limit retry budget exhausted") }
            delay := time.Second * time.Duration(1 << attempt)
            retry := res.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                candidate := time.Duration(seconds) * time.Second
                if candidate > delay { delay = candidate }
            } else if when, err := http.ParseTime(retry); err == nil {
                if candidate := time.Until(when); candidate > delay { delay = candidate }
            }
            time.Sleep(delay)
            continue
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            panic(fmt.Sprintf("HTTP %d: %s", res.StatusCode, data))
        }
        return data
    }
    panic("retry budget exhausted")
}

func discover(base, key, id string) contract {
    data := request(http.MethodGet, base + "/discovery/" + url.PathEscape(id), key, "", nil)
    var c contract
    if err := json.Unmarshal(data, &c); err != nil { panic(err) }
    if c.Method == "" || !strings.HasPrefix(c.Path, "/v1/") {
        panic("invalid discovery contract")
    }
    return c
}

func destination(data []byte, pointer string) string {
    var value any
    if err := json.Unmarshal(data, &value); err != nil { panic(err) }
    if !strings.HasPrefix(pointer, "/") { panic("expected JSON pointer") }
    for _, part := range strings.Split(pointer[1:], "/") {
        part = strings.ReplaceAll(strings.ReplaceAll(part, "~1", "/"), "~0", "~")
        object, ok := value.(map[string]any)
        if !ok { panic("pointer requires an object") }
        value = object[part]
    }
    email, ok := value.(string)
    if !ok || email == "" { panic("account destination is missing") }
    return email
}

func main() {
    base := strings.TrimRight(required("API_BASE"), "/")
    if !strings.HasSuffix(base, "/v1") { panic("base must end in /v1") }
    origin := strings.TrimSuffix(base, "/v1")
    key := required("INFRAI_API_KEY")
    account := discover(base, key, required("AUTH_CAPABILITY"))
    send := discover(base, key, required("EMAIL_CAPABILITY"))
    if send.Path != "/v1/email/send" { panic("select the email send contract") }
    if account.Method != http.MethodGet || send.Method != http.MethodPost {
        panic("expected account GET and email POST contracts")
    }
    if !send.Idempotent { panic("send contract must support idempotency") }
    var pathValues map[string]string
    if err := json.Unmarshal([]byte(required("AUTH_PATH_VALUES")), &pathValues); err != nil { panic(err) }
    path := account.Path
    for name, value := range pathValues {
        path = strings.ReplaceAll(path, "{" + name + "}", url.PathEscape(value))
    }
    if strings.ContainsAny(path, "{}") { panic("unresolved path parameter") }
    result := request(account.Method, origin + path, key, "", nil)
    email := destination(result, required("ACCOUNT_EMAIL_POINTER"))
    var payload map[string]any
    if err := json.Unmarshal([]byte(required("EMAIL_PARAMS")), &payload); err != nil { panic(err) }
    if payload == nil { panic("send parameters must be an object") }
    payload[required("EMAIL_RECIPIENT_FIELD")] = email
    body, err := json.Marshal(payload)
    if err != nil { panic(err) }
    request(send.Method, origin + send.Path, key, required("SEND_KEY"), body)
    fmt.Println("Send accepted; delivery confirmation remains separate.")
}
```

The account lookup response feeds the send payload, and every request uses the same key and base. HTTP errors retain their response body for diagnosis; run this probe in a controlled environment because those bodies can contain sensitive details. The code retries rate limits with exponential backoff and honors longer server retry instructions. Network errors stop the probe: the worker that owns the job can retry using the same operation identity.

A successful run establishes the handoff and send acceptance. It does not test inbox placement, token redemption, or an SMS fallback. That narrower claim is deliberate.

## 4. Budget the polling and suppression seam

For this shared REST option, email events are retrieved by polling rather than pushed through webhooks. That is a selection boundary for a workflow that needs immediate event-driven orchestration, and it should appear in the SLO review before anyone promises instant fallback to a seller. A missing delivery event is not evidence that the email failed.

Use a proposed capacity exercise: 600 outstanding messages polled individually every 30 seconds would require 20 checks per second before retries. Those are planning inputs, not service limits or measurements. Prefer available event-list operations where their documented semantics fit, and size the polling workers against the actual contract and rate limits. Count discovery traffic during startup, too; loading contracts once per deployed worker avoids treating discovery as a per-message dependency.

Suppression state needs explicit handling before production release. Consult the email suppression check before enqueueing another recovery email, record why the job stopped, and keep the public recovery response from disclosing account existence. The sample isolates the account-to-email seam; add suppression handling in the worker using its discovered request and response schema rather than assuming an undocumented boolean.

A polling checkpoint also needs a defined owner. For the proposed seller-42 acceptance test, keep the order identifier, recovery-attempt identifier, send operation key, and returned send result in separate fields in your application record, then let the event worker update that record using the documented event schema. If the worker restarts after observing an event but before persisting its checkpoint, processing the same event again should leave the application record in the same state. If the send request times out, keep the original operation key attached to the job instead of manufacturing another key during the next worker attempt; otherwise a retry policy that looks careful in a code review can still create a second logical send. The retention period of that job must also match the provider's deduplication contract. These are application design requirements, not assurances about indefinite server-side deduplication, and they are why a transport comparison without a recovery-job lifecycle is incomplete.

Checkpoint before fallback.

SMS fallback adds another policy decision. A blocked email address does not establish permission to send an SMS, and an account identifier does not establish a verified phone destination. Link the SMS job to the recovery attempt, resolve its destination from trusted account state, and apply channel-specific suppression rules. Geographic anti-abuse limits and country-level spending cutoffs belong in the application policy for this option.

Keep OTP ownership explicit as well: the email side does not provide managed OTP issuance. Retain that responsibility in the auth system. For a seller order alert, use an order-specific message identity and template; do not recycle the recovery message or its authorization credential.

## 5. Decide at the rollout gate

Choose the HTTP path if the beginner backend already owns message delivery, the account-to-recipient mapping is inspectable, and polling satisfies the delivery-state SLO. Common reset-link emails can use templates and single sends; campaign machinery does not remove any of these integration obligations.

Choose another transport when the auth product expects SMTP out of the box. Choose another event integration when immediate webhook-driven coordination is a hard requirement. A shared key reduces credential plumbing, but it does not merge account authorization, email suppression, and SMS permission into one decision.

The release gate should demonstrate three things with test accounts: duplicate recovery jobs preserve their operation identity, a suppressed destination does not produce repeated sends, and an unrelated order alert retains its own lifecycle. Until those checks pass, a successful API response is too small a result to call the seller workflow ready.

## References

RFC 7208 defines SPF's sender-authorization mechanism; it is useful for the sending-domain review, not evidence that a particular message reached an inbox. NIST's authentication guidance is relevant to the boundary between delivering a recovery message and authorizing account access. Provider documentation below is the starting point for checking the current integration contracts.

- [SPF, RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)
- [NIST authentication guidance](https://pages.nist.gov/800-63-3/sp800-63b.html)

## Sources

- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
