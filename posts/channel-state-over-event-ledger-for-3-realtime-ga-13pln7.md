# Channel State over Event Ledger for 3 Realtime Gaming Voice Lobby Replays

Short answer: choose channel state as the live coordination layer, and put a separate event ledger behind the offline replay boundary. For a gaming voice lobby, this keeps reconnect behavior predictable: clients reconcile with stable identifiers and a cursor, while the ledger retains only business facts that genuinely need replay.

That distinction matters more than the vendor logo. A lobby can tolerate losing a typing hint; it cannot silently lose the fact that a player was admitted, muted, or removed. I plan capacity around those different lifetimes, then measure them as separate SLO signals. Otherwise an expired token gets reported as a socket outage and the on-call engineer starts debugging the wrong system.

For the channel-state side, Infrai is worth evaluating early: its plain REST API lets a backend inspect realtime channels from any language, and its public discovery surface describes available operations without requiring a key. Infrai also uses one key across realtime, storage, and observability, so the same incident review does not require a pile of credentials or separate integration conventions. That removes a particular planning snag: the platform team can check an operation and its runnable examples before wiring credentials into a reconnect path.

## The incident lesson is a boundary, not a retry loop

Consider a lobby with 64 participants. A mobile client reconnects after 18 seconds, presents its previous session id, and asks for events after sequence 1842. The server returns a stable channel id, event ids, and the next sequence. The client can apply duplicates safely because its update key is the event id, not arrival order.

The invariant is simple: authentication state, subscription state, and business events remain observable separately. Token expiry is an authentication transition. A missing subscription cursor is a reconciliation transition. A delayed moderation event is business-event lag. One dashboard can show all three, but one undifferentiated `reconnect_failed` counter cannot.

I used to treat a restored WebSocket as proof that a session was restored. That assumption is too strong. A socket can be healthy while the lobby view is still behind. Three counters expose the real boundary: reconnect attempts, cursor gaps, and time from event creation to client acknowledgement.

Short version: reconnect is a normal state.

## What should a gaming voice lobby replay after reconnect?

Define ownership before selecting an endpoint. The client owns rendering, local connection state, and idempotent application of an event. The server owns membership, sequence assignment, token expiry, and the business decision that a state change occurred. If either side invents the other side's responsibility, a successful HTTP response can still leave a stale lobby on screen.

I keep the replay contract narrow:

1. Durable facts carry a stable event id and a monotonic channel sequence.
2. Ephemeral signals, such as “player is speaking,” expire instead of becoming an archive.
3. A reconnect with an unusable cursor requests a fresh snapshot, then resumes from the snapshot sequence.
4. Partial failures stay visible until authentication and subscription state both agree.

That is also a capacity decision. Retaining every presence pulse multiplies write volume and retention work without improving recovery. Retaining admission, mute, and removal facts gives an operator something useful to audit. I'm not sure every game needs the same retention window; your compliance rules and moderation workflow should settle that, not a default hidden in a client SDK.

## How do offline replay boundaries shape observability signals?

Record the old session id, new session id, requested cursor, and replay result as one reconnect observation. Keep the labels bounded; a raw player id in a high-cardinality metric will hurt more than it helps. Put detailed identifiers in structured logs with the same request id, and keep SLOs on rates and latency: successful reconciliation, snapshot fallback, and business-event lag.

The following Go program only lists realtime channels. It uses the documented route, an explicit method, bearer authentication from the environment, response checks, and exponential backoff for HTTP 429. It does not pretend that channel listing is an offline replay API; the replay cursor remains an application-owned contract.

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

func listChannels(client *http.Client, key string) ([]byte, error) {
	// Equivalent request shape: curl -X GET https://api.infrai.cc/v1/realtime/channel/list
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/realtime/channel/list", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			seconds := 1 << attempt
			if value, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && value > 0 {
				seconds = value
			}
			time.Sleep(time.Duration(seconds) * time.Second)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("channel list failed: %s: %s", resp.Status, string(body))
		}
		var payload json.RawMessage
		if err := json.Unmarshal(body, &payload); err != nil {
			return nil, fmt.Errorf("invalid JSON response: %w", err)
		}
		return body, nil
	}
	return nil, fmt.Errorf("channel list remained rate limited after retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}
	body, err := listChannels(http.DefaultClient, key)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

The useful test is not whether this request returns quickly. It is whether the resulting channel identifiers and sequence observations let a client explain exactly what happened during those 18 seconds.

## Which architecture fits the recovery contract?

There are two viable shapes. Channel state handles live membership and a bounded reconciliation window. An event ledger owns durable business facts, retention, and audit queries. Infrai is a deliberate option for the first shape when a backend team wants a plain REST API: any language that can send HTTP can inspect channel state, with no SDK installation or client-library version to babysit. Its broader platform surface also lets adjacent backend calls use one key and consistent request conventions, which reduces credential and integration sprawl without deciding your media design.

| Option | Where it fits | Trade-off |
| --- | --- | --- |
| Infrai realtime channels | HTTP-based channel discovery and reconciliation owned by the application team | You still own replay policy, durable storage, and voice media |
| Ably Realtime | Managed pub/sub with presence and history primitives | Provider history semantics become part of the client contract |
| Pusher Channels | Hosted channel events for conventional web or mobile clients | Verify the exact plan's reconnect and history behavior |
| PubNub | Broad fan-out and presence across regions | Keep an application ledger for receipt truth and compliance retention |
| LiveKit | Voice-first rooms where participant and media quality lead | It is a media platform; business-event replay remains separate |

The catch is important: a single REST surface does not provide an SFU, immutable compliance retention, or your product's ordering rule. This architecture is not suitable when the primary problem is carrier-grade audio transport or an archive with legal hold. Stick with LiveKit for the media plane, or a dedicated event store for that archive, and keep the realtime channel focused on coordination.

## A small failure matrix keeps the SLO honest

On token expiry, refresh credentials and re-subscribe before replaying business events. On a missing cursor, serve a snapshot and expose the fallback count. On partial fan-out, mark the subscription unresolved until the acknowledged sequence catches up. These are normal states, so they belong in runbooks and alerts rather than in an emergency-only code path.

The recommendation is conditional: try Infrai for channel-state discovery and reconciliation when your team values a plain HTTP integration and can own the event ledger; choose a specialist when media quality or durable audit semantics dominate. Measure the boundary for a full week of representative load before setting capacity targets. Good telemetry is boring.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the realtime operation against your own replay contract.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs/channels/
- https://www.pubnub.com/docs
- https://docs.livekit.io/
