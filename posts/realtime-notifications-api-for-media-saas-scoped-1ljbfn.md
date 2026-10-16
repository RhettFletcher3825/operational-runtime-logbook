# Realtime Notifications API for Media SaaS — Scoped Client Trust

The page for a media SaaS says its realtime notifications API has missed the `read_receipt_freshness` objective for ten minutes. The on-call sees a healthy Node.js application, normal publish latency, thousands of connected browser sessions, and a growing gap between the newest receipt stored by the conversation service and the newest receipt rendered by clients.

Transport is up.

Trust is not the same as correctness.

**TL;DR:** For a media SaaS adding typing indicators and read receipts without operating a WebSocket fleet, the least complex defensible option is a managed channel service behind a server-side token issuer. Give each browser a short-lived capability for one conversation, treat typing as disposable, and commit monotonic read progress through the application API before broadcasting it. Evaluate the service by authorization semantics, reconnect behavior, and observability rather than by its connection demo. The application server remains the authority; the realtime layer is a delivery path.

That answer rules out two tempting shortcuts: a broad token embedded in the browser and a design in which receipt events are the database. Both look excellent in a prototype. Neither gives an on-call a reliable way to distinguish a stalled subscriber from an unauthorized publisher or a user who merely reconnected.

## Should a SaaS notifications API let Node.js clients publish realtime events?

Work backward from the page. A receipt shown in the UI is derived from a durable fact: user `u_184` has read through item sequence `8421` in conversation `conv_7f3`. If the browser publishes `{read: 8421}` directly to a channel and every other browser trusts that event, a successful publish proves only that the channel accepted bytes. It does not prove that the sender belongs to the conversation, that the sequence exists, or that an older delayed event cannot move the marker backward.

The earlier signal should therefore have measured end-to-end progress, not transport activity. Track the age of the latest acknowledged durable receipt, split by region and client release, alongside authorization denials, reconnect attempts, subscription success, and event lag. A transport latency histogram is useful, but it cannot defend the user-visible SLO by itself.

Typing has a different contract. `typing=true` can expire after a small, explicitly chosen interval; loss is preferable to replaying stale activity after a reconnect. A read receipt cannot silently disappear, because users expect another device opened later to reconstruct read progress. One channel can carry both event types, yet they should not share durability rules.

This distinction is the first capacity-planning decision. Typing traffic grows with keystroke bursts and active participants, while durable receipt writes should be coalesced to the greatest observed sequence per user and conversation. Planning both from average messages per second conceals the burst that matters. Model active conversations, peak participants per conversation, typing emission rate, reconnect fan-out, token issuance rate, and receipt write amplification separately.

## Put authority before fan-out

The browser is an untrusted client. It may request access, but only the application backend can decide which conversation it may join and which operations it may perform. A bearer token must be protected from disclosure because possession is sufficient to use it, as RFC 6750 explains; a signed token also needs validation of its claims, rather than mere decoding, under the JWT model in RFC 7519.

A practical capability binds at least the authenticated subject, conversation identifier, allowed operations, audience, and expiration. Do not issue `publish:*` because the UI happens to need one publish operation today. Separate receipt submission from typing publication when the managed service supports operation-level permissions. If it does not, route both writes through the application backend and allow browser tokens to subscribe only. That costs an extra hop, but the trust boundary is legible.

The following Go sketch shows the policy decision, independent of any channel vendor. It intentionally emits claims rather than a service-specific token; production signing and key rotation belong in the issuer, using a maintained JOSE implementation and an algorithm fixed by policy.

```go
package capability

import (
	"errors"
	"time"
)

type Membership interface {
	CanReadConversation(userID, conversationID string) bool
}

type Claims struct {
	Subject      string   `json:"sub"`
	Audience     string   `json:"aud"`
	Conversation string   `json:"conversation"`
	Operations   []string `json:"operations"`
	ExpiresAt    int64    `json:"exp"`
}

func Authorize(m Membership, userID, conversationID string, now time.Time) (Claims, error) {
	if !m.CanReadConversation(userID, conversationID) {
		return Claims{}, errors.New("conversation access denied")
	}

	return Claims{
		Subject:      userID,
		Audience:     "realtime-gateway",
		Conversation: conversationID,
		Operations:   []string{"subscribe", "publish:typing"},
		ExpiresAt:    now.Add(5 * time.Minute).Unix(),
	}, nil
}
```

Five minutes is an example policy value, not a universal optimum. The right lifetime must be shorter than the time in which revoked access would become unacceptable, but long enough that refresh traffic and transient issuer failures do not eject healthy sessions. Measure refresh success and expiry disconnects before tightening it. Also reject tokens with the wrong audience, expired time, disallowed operation, or mismatched conversation on every channel authorization path.

Do not put secrets in query strings. Redact authorization material from logs, and make revocation behavior explicit: membership removal might block the next refresh, force an immediate disconnect, or both. Those choices create different exposure windows and different control-plane loads.

## Make read progress monotonic

The receipt write path should accept an authenticated command such as “advance my read position for this conversation to 8421.” The server verifies membership and item bounds, then stores `max(current, proposed)` transactionally. After commit, it emits the resulting position with a stable event identifier. A duplicate becomes harmless, and an out-of-order delivery cannot move progress backward.

This is state convergence, not exactly-once delivery. RFC 6455 defines the WebSocket protocol, but it does not supply application-level replay, durable storage, authorization policy, or deduplication. Those contracts still belong to the system design even when a third party owns connection termination.

On reconnect, fetch the current receipt state through the ordinary application API, establish the subscription, and reconcile any event received around that boundary by taking the greater sequence. If the channel service offers resumable streams, record the resume outcome and test the retention boundary; never assume a reconnect implies a complete history. WebRTC is also the wrong default abstraction here: its peer-connection model and data channels solve direct peer communication, while this workload needs server authority, membership enforcement, and recoverable state.

Short-lived typing events need less ceremony. Include a conversation, actor, and expiry time; rate-limit them; render only while unexpired; never backfill them. Stop events are an optimization because expiry is the correctness mechanism.

## Buy the connection plane, keep the control plane

“Managed” removes socket-server operation, not engineering ownership. The useful comparison is which responsibility moves and which remains.

| Decision | Managed channel plane | Self-hosted connection plane |
|---|---|---|
| Connection termination and fan-out | Provider operates capacity and regional edges | Team provisions, scales, drains, and patches it |
| Identity and conversation membership | Application still decides and issues scoped capability | Application still decides; enforcement is local |
| Receipt durability and ordering | Application database remains authoritative | Application database remains authoritative |
| Reconnect and replay contract | Verify provider semantics and limits | Design, implement, and operate the contract |
| On-call surface | Dependency health, quotas, token issuer, end-to-end SLO | All of those plus socket fleet and network lifecycle |
| Lock-in pressure | SDK, channel semantics, history, presence, authorization model | Protocol and storage choices are controlled internally |

My selection rule would be strict: buy the connection plane when the team does not seek differentiation there, but keep capability issuance, membership, durable receipts, and a transport adapter under application ownership. This constrains migration work to delivery semantics instead of forcing an identity rewrite.

The limitation is control. This managed approach is not suitable when the organization must own the wire protocol, place every connection endpoint inside a constrained network, or change delivery internals faster than a provider contract permits; a self-hosted connection plane is the clearer choice in those cases. The downside is a larger capacity, patching, deployment, and on-call surface. There is no free branch in this decision.

Assess candidates with a test harness, not a feature checklist. At minimum, expire a token during an active session, revoke membership, disconnect a client beyond the replay window, deliver duplicate and reversed receipt events, reconnect ten clients for one user, and rotate signing keys. Observe what the SDK reports. An API that hides resume failure behind a generic “connected” state makes incident diagnosis expensive even if its happy path is concise.

Capacity review needs provider limits expressed against workload variables: concurrent connections, peak messages, payload size, channel cardinality, token requests, and reconnect storms. Price is secondary because a technically cheap path that pages the platform team on every regional network event has transferred cost into on-call load. Conversely, self-hosting is rational when protocol control or regulated placement is material and the team can staff the resulting operational surface.

## Instrument the missing causal chain

Return to the original page. Add a receipt event identifier and the committed sequence to structured records at four boundaries: database commit, publish acceptance, subscriber receipt, and UI application. Carry a trace context where the client and transport permit it, but do not make perfect distributed tracing a prerequisite for correlation. A stable event ID plus conversation-safe hashed dimensions can answer the immediate question without logging message content or raw tokens.

Define separate indicators:

- receipt commit success and latency at the authoritative API;
- commit-to-render age for sampled clients;
- token issue and refresh outcomes by denial reason;
- subscriber resume results, including a distinct lost-history outcome;
- typing event delivery as diagnostic telemetry, not a durable-data SLO.

OpenTelemetry supplies common models for traces and metrics, while its guidance also warns that high-cardinality attributes can create operational cost. User and conversation identifiers do not belong on metric labels. Keep them in access-controlled logs or sampled traces under an explicit retention policy.

The alert should page on sustained user impact that needs immediate action, such as receipt freshness violating its objective across enough eligible sessions, and attach the causal breakdown. A token-denial increase might be a ticket when it affects a single obsolete client release; widespread refresh failures approaching token expiry are page-worthy. Tie severity to the error budget and urgency, not to the emotional force of a raw event count.

Then test the alert by fault injection in a non-production environment: deny publish permission, delay the durable write, drop the subscription, and exhaust replay retention. Confirm that the page identifies the failing boundary and that the runbook can separate issuer, storage, delivery, and client-application faults.

Keep the threshold honest. Set it too loose and users see stale receipts while every dashboard stays green; set it too tight and normal mobile suspension, sampling gaps, or a tiny cohort wakes the on-call without a useful action. False positives consume attention and train responders to distrust the page. The correct threshold comes from the stated receipt SLO, eligible-session definition, and observed baseline, followed by periodic review as client behavior changes. No managed API can choose that policy for you.

## Further reading

- WebSocket Protocol, RFC 6455: https://www.rfc-editor.org/rfc/rfc6455
- OAuth 2.0 Bearer Token Usage, RFC 6750: https://www.rfc-editor.org/rfc/rfc6750
- JSON Web Token, RFC 7519: https://www.rfc-editor.org/rfc/rfc7519
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- OpenTelemetry documentation: https://opentelemetry.io/docs/
- Google SRE Workbook, Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
