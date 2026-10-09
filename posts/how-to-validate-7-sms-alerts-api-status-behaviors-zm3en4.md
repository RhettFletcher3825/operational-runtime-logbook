# How to Validate 7 SMS Alerts API Status Behaviors for SaaS Apps

Choose an SMS alerts API for a SaaS app's short-lived password reset by testing the recovery workflow, not by counting SDK features. **The best fit is the service that passes seven application-owned integration tests with the least adapter code and produces enough polling evidence to explain an expired attempt.** A quick first send is weak evidence; the expensive part is discovering, under pressure, that acceptance, delivery, expiry, and authorization were treated as one event.

TL;DR: require a stable message identifier, poll it through a narrow internal interface, and test accepted, pending, delivered, permanent failure, unknown status, ambiguous submission, and expiry behavior before choosing an integration. Run the same suite against US and EU test destinations. Keep reset validity in the account system, because transport status must never extend a credential's lifetime.

## Can a SaaS App Trust SMS Alerts API Status Polling?

In a production-readiness review, I start with one bounded incident: a fintech user requests a reset, the send call returns without a conclusive delivery result, and the short expiry passes while polling still reports a nonterminal state. The useful question is not which dashboard looks friendliest. It is: can an operator determine whether the application submitted one message, whether another submission is safe, and whether any arriving code is still valid?

Those answers require separate records. The account service owns the reset attempt, its expiry, single-use enforcement, and the decision to authorize a password change. The messaging adapter owns an opaque external identifier and a normalized observation of transport state. A status such as `delivered` can describe transport evidence; it cannot renew an expired reset or prove that the intended account holder read the message.

This separation matters for security as well as operations. NIST SP 800-63B describes use of the public switched telephone network for out-of-band authentication as restricted and directs verifiers to consider risk indicators including device swaps and number porting. For a fintech recovery flow, SMS therefore belongs inside a risk decision with rate controls and another recovery route, rather than acting as unquestionable proof of identity.

The invariant is blunt. Expiry wins.

## Turn the selection into seven executable checks

An evaluation should begin with the smallest contract the application needs: submit a transactional notification and inspect the resulting message. Every candidate adapter then faces the same seven cases. This changes integration effort from a vague impression into code that reviewers can read and rerun. A Node.js SaaS app can own this policy while a Go adapter implements the messaging boundary; the runtime split doesn't change the tests.

| Check | Stimulus | Required application behavior |
|---|---|---|
| 1. Accepted | Submission returns an external ID | Persist the ID against one reset attempt |
| 2. Pending | Polling returns a documented nonterminal state | Keep waiting only inside the reset window |
| 3. Delivered | Polling returns a documented terminal success | Record transport evidence without changing expiry |
| 4. Permanent failure | Polling returns a documented terminal failure | Stop polling and offer the approved recovery path |
| 5. Unknown status | The adapter receives an unrecognized value | Preserve it for diagnosis; never map it to success |
| 6. Ambiguous submission | The request outcome is unknown | Reconcile before allowing another send |
| 7. Expired | The deadline passes first | Stop the recovery attempt regardless of later delivery |

Do not invent the external state names during evaluation. Take them from each candidate's current documentation, map them once inside the adapter, and keep the raw value in restricted diagnostic data. The application should see only the states it can act on.

The following Go program is a compact conformance harness for that boundary. It is deliberately a fake-backed executable, not vendor setup code; replace `scriptedGateway` with each adapter and retain the same assertions.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

type State string

const (
	Pending   State = "pending"
	Delivered State = "delivered"
	Failed    State = "failed"
	Unknown   State = "unknown"
)

var ErrExpired = errors.New("reset expired")

type Receipt struct {
	ID    string
	State State
}

type Gateway interface {
	Send(context.Context, string, string) (Receipt, error)
	Status(context.Context, string) (Receipt, error)
}

type scriptedGateway struct {
	states []State
	next   int
}

func (g *scriptedGateway) Send(context.Context, string, string) (Receipt, error) {
	return Receipt{ID: "test-message", State: Pending}, nil
}

func (g *scriptedGateway) Status(context.Context, string) (Receipt, error) {
	if g.next >= len(g.states) {
		return Receipt{ID: "test-message", State: Unknown}, nil
	}
	s := g.states[g.next]
	g.next++
	return Receipt{ID: "test-message", State: s}, nil
}

func reconcile(ctx context.Context, g Gateway, id string, expiresAt time.Time) (Receipt, error) {
	for time.Now().Before(expiresAt) {
		r, err := g.Status(ctx, id)
		if err != nil {
			return Receipt{}, err
		}
		if r.State == Delivered || r.State == Failed {
			return r, nil
		}
		if r.State == Unknown {
			return r, fmt.Errorf("unmapped delivery state")
		}
		time.Sleep(time.Millisecond)
	}
	return Receipt{ID: id, State: Unknown}, ErrExpired
}

func main() {
	ctx := context.Background()
	g := &scriptedGateway{states: []State{Pending, Delivered}}
	r, err := g.Send(ctx, "+15555550100", "Reset code expires shortly")
	if err != nil || r.ID == "" {
		panic("accepted submission must return an ID")
	}
	r, err = reconcile(ctx, g, r.ID, time.Now().Add(time.Second))
	if err != nil || r.State != Delivered {
		panic("pending must transition to delivered")
	}

	expired := &scriptedGateway{states: []State{Pending}}
	_, err = reconcile(ctx, expired, "expired-message", time.Now())
	if !errors.Is(err, ErrExpired) {
		panic("expiry must terminate reconciliation")
	}
	fmt.Println("contract checks passed")
}
```

The millisecond wait is test scaffolding, not a production polling recommendation. Production intervals must be derived from the reset deadline, documented request limits, expected bursts, and the delay the recovery SLO permits. That calculation belongs in the review because polling consumes capacity even when no status changes.

## Make ambiguity more expensive than extra adapter code

The hardest case is a submission whose outcome is unknown: the caller lost the response, but the remote system may have accepted the message. Blindly sending again can create two valid-looking texts, possibly delivered out of order. A candidate that supports an application-supplied idempotency mechanism can simplify this branch; otherwise the adapter needs a documented reconciliation rule and the account service must prevent uncontrolled retries. The relevant comparison is the amount of application code and operational judgment needed to reach a deterministic result.

Test this boundary by interrupting the client after request transmission and before response processing. Then verify what durable evidence remains, how the attempt is reconciled, and what an operator sees. Do the same with an invalid destination, an authentication failure, a request-limit response, and a status value introduced outside the adapter's known mapping. These are evaluation cases, not claims that every API uses identical errors.

Observability should follow the reset attempt rather than the provider request. Record a correlation ID, submission outcome, normalized state, state age, poll count, and expiry result; exclude reset secrets from logs and metric labels. Break SLO evidence out by US and EU destination group, since the question is regional fitness, but validate the precise region classification and data-handling obligations with the organization's legal and security owners.

Capacity planning is straightforward once the candidate's documented constraints are known. Estimate peak concurrent reset attempts, multiply by the proposed polls per attempt, include retries, and compare the result with both request limits and worker capacity. Use a load test to settle unknowns. A design that passes one manual send but exhausts its poll budget during a recovery burst has not passed integration.

## Decide which responsibility the team is buying

No selection removes ownership; it moves it. A managed messaging connection can reduce carrier-facing integration work, while the application still owns recovery policy and audit evidence. Building a broader internal layer can reduce coupling at the cost of more code, testing, and on-call surface.

| Boundary | Buy more of the capability | Build more of the capability |
|---|---|---|
| Carrier and regional connectivity | External service owns more connectivity work | Team owns more direct integration work |
| Status vocabulary | Adapter maps one external contract | Internal layer maps several contracts |
| Routing and failover | Depend more on the purchased service | Operate routing logic and its evidence |
| Recovery authorization | Never delegate to transport status | Keep in the account service |

I would score integration effort from the proof artifacts: adapter branches, contract fixtures, deployment configuration, background-worker behavior, security review items, and runbook decisions. SDK brevity gets little weight. Price can be recorded for planning, but it does not resolve ambiguous submission or reduce the number of failure branches an on-call engineer must understand.

The limitations are material. This method isn't suitable when the threat model rules out SMS recovery, when users cannot reliably access a mobile number, or when status latency must be lower than a polling design can provide. Polling also trades delayed state observation and repeated requests for a simpler inbound network boundary; an authenticated, deduplicated webhook consumer is the alternative when prompt updates justify that extra receiver. In the other cases, change the recovery channel or the event transport. **For a short-expiry reset that intentionally avoids webhooks, select only after the seven tests pass in the real deployment regions and the calculated polling load fits the service limits.**

## Sources

References:

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
