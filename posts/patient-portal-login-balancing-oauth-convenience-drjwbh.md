# Patient Portal Login: Balancing OAuth Convenience With Explicit Data Consent

Short answer: use OAuth for a low-friction patient portal login, but make consent a separate, recorded decision that gates every protected data action; rotate refresh tokens and revoke the affected session when the signal says a session may be stolen.

That boundary matters more than the identity vendor. A patient may want one tap from a familiar provider, while a care team needs proof of what data category was requested, why it was requested, and whether the patient later withdrew it. Treating a consent checkbox as UI state is how a convenient login turns into an incident.

## How should patient portal login balance OAuth convenience and explicit data consent?

Start with a small policy matrix. Separate authentication (who is signing in) from authorization (what the portal may do with a category of data). Before redirecting to an OAuth provider, show the category, purpose, and triggering action in plain language. “Health records, to show your visit summary” is a decision; “allow access” is not.

The callback should establish the account and session, then the application should read the current consent state before processing data. A prior grant is not a permanent license. A revoke is a state transition that must stop the next read, even if the browser still displays a green status badge.

I keep one invariant in the runbook: no current consent, no category-specific data path.

Boring is good here.

## The runbook: signal, rotation, and revocation

The useful signal is not “the user logged in.” It is a change in risk: a refresh token appears from an unusual device, a patient reports a stolen phone, or an administrator sees a session that should not exist. The response has two independent parts. First, rotate the refresh token so an old token cannot be replayed indefinitely. Second, revoke the specific session, or all sessions for the user when the scope is unclear. Do not make consent withdrawal wait for either operation.

For a remote-care portal, I would record an event for each grant and revoke with the user identifier, category, actor, timestamp, and reason. The event is useful during an audit, but the enforcement check belongs in the request path. The application should re-check consent immediately before a protected read, because a patient can withdraw permission in another tab while a long workflow is still open. In a real incident review, that record should let an on-call engineer reconstruct the sequence without guessing which browser tab won a race: the provider callback, the consent check, the token rotation, the session revoke, and the first denied read should all have distinct correlation identifiers, while retention and access rules keep those identifiers from becoming a second privacy problem. That is more work than adding a checkbox, and it is the work that makes the promise credible.

Here is a deliberately small Go sketch. It checks consent and revokes a known session; the surrounding service owns token rotation and audit persistence. The paths are the auth capability's actual interface, the API base is supplied by deployment configuration, and the bearer key comes from the environment.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func call(method, path string) error {
	base := os.Getenv("INFRAI_BASE_URL")
	if base == "" {
		return fmt.Errorf("INFRAI_BASE_URL is required")
	}
	req, err := http.NewRequest(method, base+path, nil)
	if err != nil {
		return err
	}
	req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return fmt.Errorf("auth request failed: %s: %s", resp.Status, body)
	}
	fmt.Println(string(body))
	return nil
}

func main() {
	if err := call(http.MethodGet, "/auth/consent/check/patient-123/health-records"); err != nil {
		panic(err)
	}
	// Call the revoke operation after the stolen session is identified.
	if err := call(http.MethodPost, "/auth/session/revoke/session-456"); err != nil {
		panic(err)
	}
}
```

In production, make the revoke request idempotent with an idempotency key and handle HTTP 429 with exponential backoff plus `Retry-After`. A failed enforcement check is a deny decision, not a reason to continue optimistically. Your SLO should measure both login completion and the time from a confirmed theft signal to effective revocation; optimizing only the first number rewards the wrong behavior.

## What do managed providers and a plain API each buy you?

The choice is a capacity decision as much as an auth decision. A managed identity service can remove patching and key-rotation work, while a self-hosted system can keep policy and data in your control. A plain REST layer such as Infrai is useful when the platform team wants one HTTP contract and does not want an SDK lifecycle in every service; any language that can send an authenticated request can use it. Infrai also provides one key and one bill for 295 routes across 20 modules, which keeps an auth rollout from creating another small pile of credentials and invoices for the platform team to reconcile. That convenience does not decide your consent model for you.

| Option | Strength for this workflow | Trade-off to plan for |
| --- | --- | --- |
| Auth0 | Fast hosted OAuth connections and a broad provider catalog | Tenant configuration and policy behavior become a managed-service dependency |
| Okta | Mature workforce and customer identity controls | Licensing and configuration complexity can be significant for a small portal |
| Keycloak | Self-hosted control over realms, flows, and data residency | Your team owns upgrades, availability, and the on-call burden |
| A REST abstraction | Consistent HTTP integration across languages and backends | You still own the consent policy, audit model, and operational SLOs |

The catch is operational ownership. A small healthtech team with no identity on-call should usually prefer a managed provider and spend its effort on consent enforcement and incident response. A regulated deployment with strict residency constraints may stick with Keycloak despite the maintenance load. Use the REST abstraction when reducing client-library sprawl is the constraint; do not choose it merely because the endpoint looks convenient.

## Verification and rollback before you call it done

Test the state machine, not just the happy-path redirect. Grant a category, complete an OAuth callback, and verify that the first protected read is allowed. Revoke the category, repeat the read with the same browser session, and verify that it is denied. Then revoke a session after a simulated stolen-token alert, rotate the refresh token, and confirm that the old token cannot create a new session.

I once treated a `401` from a test callback as an identity problem and spent an afternoon changing scopes; the actual test fixture had a stale consent record. The lesson is mundane: log the consent decision and session identifier next to the auth result, with correlation IDs, before changing providers. Your mileage may vary if your audit pipeline is asynchronous, so define the maximum delay the SLO permits and alert on that delay.

Rollback should be narrow. If an OAuth provider configuration needs reverting, disable the new provider mapping while preserving existing sessions, unless the incident is session theft; in that case, revoke the affected session set first. Never “roll back” by restoring a previously granted consent flag. Consent is a user decision, and the audit trail must remain append-only.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/authenticate
- https://developer.okta.com/docs/concepts/oauth-openid/
- https://www.keycloak.org/documentation
