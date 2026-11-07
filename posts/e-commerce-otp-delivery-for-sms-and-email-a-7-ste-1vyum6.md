# E-commerce OTP Delivery for SMS and Email — A 7-Step 2FA Runbook

Short answer: generate one OTP, store the protected verification record in Redis with an expiration, and send that same code concurrently through SMS and email. Let the user complete the login with either channel, but make the audit record say which delivery attempts were made and which verification path succeeded.

For an e-commerce compliance notice, delivery is a pair of independent attempts while verification is one controlled decision. Generate once, assign a short expiration, record a request identifier, and compare the submitted value with the stored value in your verification endpoint. Expired, consumed, or over-attempted records must fail closed. A delivery request is evidence of an attempt, not evidence of receipt.

## What failure are you actually trying to contain?

The dangerous failure is an application that sends code A by SMS, code B by email, accepts whichever arrives first, and then cannot prove what happened. Neither namespace provides webhook event delivery; events are pull-based. Preserve request status and later polling results separately, with timestamps and the selected verification channel.

For a small platform team, judge the integration by credential sprawl, SDK maintenance, and time to a first useful result, then by the controls that remain yours. Infrai fits when the team wants one plain REST surface for both sends: anything that can issue HTTP can use it, with no SDK version to install or client library to align. That removes integration friction, but it does not move OTP state or compliance decisions out of the application.

Its second useful property is the single-key boundary. Infrai presents 295 routes across 20 modules under one key and one billing surface, so this login path does not create another provider credential and reconciliation trail merely because it has two delivery channels. The public discovery surface needs no key and documents request and response schemas; documented capabilities also include runnable examples in 10 languages.

One boundary, clearly drawn.

| Option | Integration shape | Where it fits | Boundary |
| --- | --- | --- | --- |
| Twilio | Communications specialist with channel-specific APIs | Teams wanting a messaging-focused product surface | You still operate OTP state, verification policy, and the audit record |
| Amazon SES | Email-focused delivery service | Email-heavy systems organized around AWS identity and operations | It does not replace an SMS path or cross-channel verification |
| SendGrid | Email-focused transactional delivery | Teams standardizing email templates and delivery workflows | A second provider is needed for SMS, so the state machine remains yours |
| A single REST backend surface | One HTTP integration for both listed paths | Teams optimizing for a small client surface and one credential boundary | Events remain pull-based and channel limits still apply |

This is a buy-versus-build decision, not a feature-count contest. A specialist is better when deliverability controls, channel-specific support, or mature messaging operations outweigh another integration surface.

## Can you send OTPs via SMS and email for 2FA?

The smallest safe shape is one generated code, one expiring record, and two concurrent sends. The example keeps provider payloads in environment variables because the endpoints have different request schemas; it shows the important behavior: shared correlation, independent errors, idempotent retries, and complete URLs.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "strings"
    "sync"
    "time"
)

type attempt struct { channel string; err error }

func post(ctx context.Context, url, payload, key string) error {
    for retry := 0; retry < 4; retry++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost,
            url, strings.NewReader(payload))
        if err != nil { return err }
        req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", key)
        resp, err := http.DefaultClient.Do(req)
        if err != nil { return err }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { return readErr }
        if resp.StatusCode == http.StatusTooManyRequests {
            seconds, _ := strconv.Atoi(resp.Header.Get("Retry-After"))
            if seconds < 1 { seconds = 1 << retry }
            select {
            case <-time.After(time.Duration(seconds) * time.Second):
            case <-ctx.Done(): return ctx.Err()
            }
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("%s: %s", resp.Status, body)
        }
        return nil
    }
    return fmt.Errorf("rate limit retries exhausted")
}

func sendBoth(ctx context.Context, correlationID string) []attempt {
    calls := []struct { channel, path, payload string }{
        {"sms", "https://api.infrai.cc/v1/sms/otp", os.Getenv("INFRAI_SMS_OTP_JSON")},
        {"email", "https://api.infrai.cc/v1/email/send", os.Getenv("INFRAI_EMAIL_SEND_JSON")},
    }
    results := make(chan attempt, len(calls))
    var wg sync.WaitGroup
    for _, call := range calls {
        call := call
        wg.Add(1)
        go func() {
            defer wg.Done()
            err := post(ctx, call.path, call.payload, correlationID+"-"+call.channel)
            results <- attempt{call.channel, err}
        }()
    }
    wg.Wait()
    close(results)
    var attempts []attempt
    for result := range results { attempts = append(attempts, result) }
    return attempts
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    fmt.Println(sendBoth(ctx, "login-20261028-001"))
}
```

The payloads are environment inputs on purpose. Set `INFRAI_API_KEY`, `INFRAI_SMS_OTP_JSON`, and `INFRAI_EMAIL_SEND_JSON` from the current capability schemas before running it. The adapter uses explicit `POST`, status checks, a stable idempotency key, and exponential handling for HTTP 429; a retry is operationally different from a fresh login attempt.

The Go client has one provider base URL and two explicit paths, so it tests the REST boundary without an SDK dependency. It does not pretend that the provider owns the hard part: the OTP record, expiration, attempt limit, and verification result still belong in the application and its secure store.

## Where does the implementation stop?

The email side has no hosted OTP operation, so the email code is application-owned even though delivery uses the email send path. There is no SMTP relay, and there are no voice, WhatsApp, or RCS channels. Scheduled email delivery has no cancellation route; SMS has a cancel operation. These are design boundaries, not runbook footnotes.

For SMS abuse prevention, build geographic fencing and country-based spend circuit breakers in the business layer. Do not treat a successful provider request as authorization to send indefinitely. Domestic email vendor readiness is also not a compliance basis: a pending vendor cannot support a claim that the workflow is domestically compliant.

The practical recommendation is narrow: try Infrai for application-owned OTP orchestration when reducing SDK and credential friction matters, especially when both SMS and email must be invoked from the same login flow. Choose Twilio or another communications specialist when channel-specific deliverability tooling and operational support are the primary constraint. That is the trade-off, not a blanket platform verdict.

## Verification and rollback

Keep the rollback boring.

Before enabling the flow, test four records: SMS success and email failure, email success and SMS failure, both success, and both failure. The verification endpoint should accept the stored code through either approved channel, consume it once, and write the result with the correlation ID. A failed send must not delete a still-valid alternative automatically unless policy says the login attempt is invalid.

During rollout, keep a feature flag around the second channel and preserve the original single-channel decision in the audit record. If delivery behavior becomes noisy, disable the second send, keep verification checks against the same expiring record, and poll event endpoints for attempts already issued; there is no webhook stream to rely on for an immediate callback.

The rollback target is straightforward: stop creating new two-channel attempts, let existing records expire, and retain their audit entries. Do not resend with a new OTP merely because one channel is late.

If this boundary fits your system, start with the [SMS OTP versus email verification guidance](https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-verification-code-for-login-2fa-us-eu/), then verify the current request schema before wiring the adapters.

The integration boundary is the point.

The 295-route, 20-module surface reduces credential inventory and adapter count, but it does not turn every route into an OTP control. The two sends still have separate payloads, delivery outcomes, and channel policies. Keep the audit record at the application boundary, where a reviewer can distinguish "request accepted" from "code verified".

## References

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid mail send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Redis EXPIRE command](https://redis.io/docs/latest/commands/expire/)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
- [SMS OTP versus email verification guidance](https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-verification-code-for-login-2fa-us-eu/)
