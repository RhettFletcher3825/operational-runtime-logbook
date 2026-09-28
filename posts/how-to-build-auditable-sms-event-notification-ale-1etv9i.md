# How to Build Auditable SMS Event Notification Alerts with Delivery Status Polling

A five-minute offer is useless if its text arrives in six minutes, and it is indefensible if the booking team cannot prove why a number was contacted. **Short answer:** model every fitness-class waitlist offer as a durable state machine, apply country and consent policy before enqueueing, treat delivery receipts as hints rather than truth, poll only while the offer remains actionable, and permit another attempt only through an idempotent transition with a fixed budget. For an e-commerce booking flow that may send a short-lived password-reset message before checkout, keep authentication and waitlist purposes separate even when they share transport infrastructure.

Set the target before writing code: 99.9% of eligible offers should reach a terminal internal state before their five-minute reservation expires, while every intentional contact has a recorded policy decision. The first is an availability SLO. The second is a compliance invariant, not an average.

## How should Node.js SMS event notification alerts handle delivery status?

Consider a bounded incident. A class has one newly open place. The service texts the first eligible member, status polling stalls, and an operator creates another attempt. The original message then arrives. Two people can see apparently valid links for one place, support cannot tell which attempt was authorized, and a cancellation races both messages. No provider outage is required; ordinary queueing, delayed carrier receipts, and an unsafe retry policy are enough.

The invariant is stricter than "send once." At most one active offer may exist for a class seat, every transport attempt must belong to that offer, and accepting the seat must be a conditional database transition. A receipt cannot reserve inventory. A successful submission cannot prove handset delivery. Cancellation must invalidate the claim token even when transport cancellation is no longer possible.

One seat. One winner.

The dangerous timeline is worth spelling out because dashboards tend to flatten it into a single red or green row. At 10:00:00 the seat-release transaction creates offer A with version 1 and a 10:05:00 expiry. At 10:00:02 the worker records attempt 1, but its submission response is lost after the downstream service accepts the request. At 10:00:12 another worker sees no receipt. It must not infer that nothing was sent; it first reloads offer A, checks the attempt budget and expiry, and conditionally advances the version. If cancellation arrives at 10:00:13, only one of those writes can win. The losing worker drops its work. If the original text appears at 10:00:20, the claim endpoint reads current business state rather than trusting the age or contents of the text. That database comparison is the control that protects inventory; receipt polling merely improves the operator's picture.

Capacity matters. Polling 100,000 pending messages every second creates 100,000 reads per second before booking traffic is counted. Back off, add jitter, and stop on a terminal state, cancellation, or expiry.

## Build the offer as a state machine

Use separate records for the business offer and each transport attempt. The offer owns eligibility, expiry, and seat assignment; an attempt owns destination, an opaque message ID, submission time, and observed delivery state. Normalize phone numbers, but never infer consent, residency, or lawful purpose from a country calling code.

```go
package waitlist

import (
    "errors"
    "time"
)

type OfferState string

const (
    Offered OfferState = "offered"
    Accepted OfferState = "accepted"
    Cancelled OfferState = "cancelled"
    Expired OfferState = "expired"
)

type Offer struct {
    ID, ClassID, MemberID string
    State OfferState
    ExpiresAt time.Time
    AttemptCount int
    Version int64
}

func (o Offer) MayTryAgain(now time.Time, limit int) error {
    if o.State != Offered {
        return errors.New("offer is not active")
    }
    if !now.Before(o.ExpiresAt) {
        return errors.New("offer has expired")
    }
    if o.AttemptCount >= limit {
        return errors.New("attempt budget exhausted")
    }
    return nil
}
```

`Version` supports optimistic concurrency: accepting, cancelling, expiring, or creating another attempt includes the version read and increments it. Exactly one contender wins. The link should carry an opaque, single-purpose token whose server-side record points to the offer and expires with it. Do not place a phone number or entitlement directly in a bearer token merely because encoding is convenient.

Issue a different single-purpose token for password reset. Its short lifetime looks similar, but it authorizes an identity action instead of scarce inventory, so its abuse controls, redaction, and audit trail must stay distinguishable.

## Poll and cancel without widening the race

A transport adapter needs a narrow contract. Submission returns an opaque ID; lookup returns a normalized state; cancellation is best-effort. The database remains authoritative.

```go
package waitlist

import (
    "context"
    "time"
)

type DeliveryState string

const (
    Pending DeliveryState = "pending"
    Delivered DeliveryState = "delivered"
    Undelivered DeliveryState = "undelivered"
)

type Messenger interface {
    Send(context.Context, string, string, string) (string, error)
    Status(context.Context, string) (DeliveryState, error)
    Cancel(context.Context, string) error
}

func NextPoll(now, expires time.Time, checks int) (time.Time, bool) {
    delays := []time.Duration{2, 5, 10, 20, 30}
    if !now.Before(expires) {
        return time.Time{}, false
    }
    if checks >= len(delays) {
        checks = len(delays) - 1
    }
    next := now.Add(delays[checks] * time.Second)
    return next, next.Before(expires)
}
```

Those intervals are an example policy, not a carrier guarantee. Choose them from reservation lifetime, observed receipt latency, provider quotas, and the maximum stale-work budget. Add jitter so a queue recovery does not synchronize every poller.

Create another attempt only after a normalized failure that policy declares retryable, or after a bounded no-receipt interval with enough useful life left. Before sending, atomically increment the count and write an outbox event; a worker submits it. This transactional boundary prevents a database commit from being separated from an unrecorded send.

Cancellation is asymmetric. First make the offer unclaimable, then request transport cancellation and record the result. Even if the text is already on a handset, its link fails closed.

## Put country policy ahead of the queue

A country guardrail should return a decision plus evidence, not a Boolean. Inputs include purpose, destination country, consent record and timestamp, suppression status, local sending window, approved sender type, and policy version. The result is `allow`, `deny`, or `manual_review`, with stable reason codes.

EU and US rules are not interchangeable. The EU ePrivacy Directive addresses unsolicited communications and member states transpose it into national law; GDPR principles such as purpose limitation and data minimization still apply to personal-data processing. In the US, the Telephone Consumer Protection Act and FCC rules are relevant, while state law and carrier requirements may add constraints. Legal counsel and the organization's documented policy must define executable rules.

Rate limits need three scopes: per destination to contain harassment and loops, per class to contain a faulty job, and global throughput to remain within tested capacity. Password-reset requests need additional per-account and per-network abuse controls, with responses that do not disclose whether an account exists. Keep raw numbers in the delivery path only for a defined retention period; use a keyed destination fingerprint for correlation.

The gate should emit an immutable record containing offer ID, purpose, policy version, consent reference, destination country, decision, reason codes, template version, and timestamp. Template ID plus variables can establish what was rendered without logging message bodies. Reset tokens must never enter logs.

## Choose ownership by evidence quality

Transport is secondary to whether the organization can operate and audit it.

| Model | Evidence boundary | On-call load | Coupling | Suitable condition |
|---|---|---:|---:|---|
| Managed messaging API | Internal decisions plus exported submission and receipt records | Lower transport burden; reconciliation remains | Adapter semantics and receipt taxonomy | Evidence exports pass a reconstruction drill |
| Cloud communications service | Application evidence plus cloud audit and delivery records | Shared with the cloud platform team | Identity, event, and regional controls | The cloud control plane is already audited |
| Self-hosted gateway | Almost the entire chain is internal | Highest; routing and upgrades join the pager | Carrier operations | Specialist staffing and volume justify ownership |

Do not score those rows by feature count. Pick an offer ID and reconstruct eligibility, consent, policy evaluation, template, attempt history, receipts, cancellation, token use, and final seat assignment without ad hoc production queries. Then work backward from a destination fingerprint to every permitted contact inside the retention window. Missing joins are defects.

If an agent can invoke messaging tools, keep its tool definition narrow and validate inputs server-side. Tool selection does not replace consent checks, authorization, or rate limits. Accept a business operation ID and template variables, then let the same policy gate and outbox decide whether transport work exists.

## Operate the SLO and recognize the boundary

Measure offer outcomes rather than submission uptime alone: time from seat release to policy decision, queue delay, submission latency, time to terminal receipt, claim success before expiry, duplicate-attempt rate, and denials by reason. Alert on error-budget burn and on any compliance-invariant breach. Preserve the raw receipt taxonomy beside the normalized state so investigations retain detail.

Test the ugly boundaries. A test adapter can delay receipts past expiry, return duplicate and out-of-order callbacks, fail cancellation, and make two workers create attempts concurrently. Property tests should assert that no transition produces two active offers for one seat and that expired or cancelled tokens never claim inventory. Start deployment with shadow policy evaluation, then a small destination cohort, comparing old and new decisions before increasing traffic.

This design is excessive for a non-urgent informational text where no scarce resource, security action, or regulated evidence depends on delivery. A queued notification with suppression checks may be enough. It is insufficient for emergency communications, where acknowledgement, escalation, and specialized obligations require a separate system.

For short-lived fitness waitlist and password-reset messages, build the durable business state and compliance evidence internally; treat transport as replaceable and probabilistic. A delayed receipt should spend an error budget, not create a second entitlement.

## References

- https://eur-lex.europa.eu/eli/dir/2002/58/oj
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://www.fcc.gov/general/telemarketing-and-robocalls
- https://owasp.org/www-community/attacks/Transaction_Logging
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
