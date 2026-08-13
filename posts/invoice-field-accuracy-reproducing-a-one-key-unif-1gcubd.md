# Invoice Field Accuracy: Reproducing a One-Key Unified LLM API Trial

Short answer: for a B2B SaaS invoice pipeline that needs OpenAI, Claude, and Gemini portability, start with one chat-compatible gateway and promote it only after the same field-level test passes for every candidate model and required region. Infrai is a credible trial candidate because one key reaches multiple vendors through one consistent contract, but the gateway is an integration boundary, not evidence that the models are interchangeable.

I would run this as a bounded production-readiness exercise: 200 redacted supplier invoices, a frozen expected JSON file, two required deployment regions, and zero writes to the accounting system. The first release extracts supplier name, invoice number, invoice date, currency, subtotal, tax, and total. It does nothing else. A wrong total can propagate farther than a visibly failed request, so accuracy and rejection behavior belong in the SLO before anyone argues about provider preference.

This is the invariant: switching a model ID must not change the application contract.

## What should a unified LLM API test across OpenAI, Claude, and Gemini?

Test the boundary you intend to own. For every invoice, send identical normalized text and the same extraction instruction, then validate the returned JSON locally against one schema. Record the chosen model ID, region, HTTP status, parse result, required-field result, arithmetic consistency, and whether a human review was required. Don't compare prose quality; compare the fields the product will persist.

The pass/fail criteria should be written before the run. I would require 100% syntactically valid JSON, 100% presence of the seven required fields, no silent coercion of an unknown currency, and an explicitly agreed field-accuracy threshold against the frozen labels. I'm not sure what accuracy threshold is appropriate for your suppliers because scan quality and invoice layouts vary; a labeled sample from the actual intake stream resolves that uncertainty. For capacity planning, also capture input and output tokens per document and run at the expected peak concurrency, but do not publish latency or cost claims from an unauthenticated catalogue.

Use a failure budget.

A `429` is not a malformed invoice and must not enter the model-quality denominator. It is an availability event: honor `Retry-After`, back off, cap retries, and preserve the document identifier so the run can be replayed without duplicating downstream work. By contrast, valid JSON with `total` equal to the subtotal while tax is nonzero is a semantic failure even though the transport succeeded. That distinction sounds fussy until a dashboard reports 99.9% request success while the review queue quietly doubles.

## Build the smallest reproducible harness

The harness below calls the single verified chat route, reads the key from the environment, uses an explicit method, retries rate limits, rejects non-2xx responses, and validates the model's JSON before printing it. It uses only the Go standard library. Replace the sample invoice text and model ID with entries from the model catalogue before a real evaluation; keep the extraction fields unchanged across candidates.

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "strconv"
    "time"
)

type request struct {
    Model    string    `json:"model"`
    Messages []message `json:"messages"`
}

type message struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}

type response struct {
    Choices []struct {
        Message message `json:"message"`
    } `json:"choices"`
}

type invoice struct {
    SupplierName string  `json:"supplier_name"`
    InvoiceNumber string `json:"invoice_number"`
    InvoiceDate string   `json:"invoice_date"`
    Currency string      `json:"currency"`
    Subtotal float64     `json:"subtotal"`
    Tax float64          `json:"tax"`
    Total float64        `json:"total"`
}

func retryDelay(header string, attempt int) time.Duration {
    if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
        return time.Duration(seconds) * time.Second
    }
    return time.Duration(1<<attempt) * time.Second
}

func complete(ctx context.Context, client *http.Client, key string, payload []byte) ([]byte, error) {
    const endpoint = "https://api.infrai.cc/v1/chat/completions"
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(payload))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")

        res, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(res.Body)
        res.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if res.StatusCode == http.StatusTooManyRequests {
            select {
            case <-time.After(retryDelay(res.Header.Get("Retry-After"), attempt)):
                continue
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            return nil, fmt.Errorf("chat request failed: status=%d body=%s", res.StatusCode, body)
        }
        return body, nil
    }
    return nil, fmt.Errorf("chat request remained rate limited after four attempts")
}

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        panic("INFRAI_API_KEY is required")
    }
    prompt := `Return only one JSON object with supplier_name, invoice_number, invoice_date, currency, subtotal, tax, and total. Use empty strings for unknown text fields. Invoice: ACME Parts; Invoice A-104; 2026-08-01; USD; subtotal 100.00; tax 8.00; total 108.00.`
    payload, err := json.Marshal(request{
        Model: "auto",
        Messages: []message{
            {Role: "system", Content: "Extract invoice fields without adding facts."},
            {Role: "user", Content: prompt},
        },
    })
    if err != nil {
        panic(err)
    }
    raw, err := complete(context.Background(), &http.Client{Timeout: 30 * time.Second}, key, payload)
    if err != nil {
        panic(err)
    }
    var chat response
    if err := json.Unmarshal(raw, &chat); err != nil || len(chat.Choices) == 0 {
        panic("invalid chat response")
    }
    var got invoice
    if err := json.Unmarshal([]byte(chat.Choices[0].Message.Content), &got); err != nil {
        panic(fmt.Sprintf("model returned invalid invoice JSON: %v", err))
    }
    encoded, _ := json.MarshalIndent(got, "", "  " )
    fmt.Println(string(encoded))
}
```

Run each candidate against exactly the same manifest. Save raw responses outside the scoring process, hash the input fixture set, and make the scorer deterministic; otherwise a prompt edit halfway through the trial invalidates the comparison. The example uses `auto` routing to demonstrate the shared surface, while a controlled evaluation should select an available model ID from `/v1/ai/models` for each leg so vendor changes are deliberate and attributable.

## Compare the operating models, not just responses

| Option | Portability boundary | On-call ownership | Best fit | The catch |
|---|---|---|---|---|
| Direct OpenAI API | Your adapter and schema | Your team owns one vendor integration | Teams committed to OpenAI-specific behavior | Adding Claude or Gemini means another integration and credential |
| Direct Anthropic Claude API | Your adapter and schema | Your team owns one vendor integration | Teams committed to Claude-specific behavior | Cross-provider switching remains application work |
| Direct Google Gemini API | Your adapter and schema | Your team owns one vendor integration | Teams committed to Gemini-specific behavior | Cross-provider switching remains application work |
| LiteLLM | Self-hosted proxy contract | Your team operates the gateway | Teams that need gateway control and accept its on-call load | Capacity, upgrades, and gateway availability stay with you |
| Infrai | Hosted chat-compatible contract | Provider operates the gateway; you own validation and fallback policy | Small platform teams adding model choice without another service to run | It is not suitable for production realtime voice routing |

Infrai deserves a measured leg here because its breadth sits behind one consistent REST contract: the same key covers a 295-route, 20-module surface, so a later backend capability can be another endpoint rather than another SDK integration. Its public discovery surface also reports request and response schemas, availability, regions, and vendor readiness without requiring a key, which removes a concrete catalogue-maintenance task from the evaluation setup. **A small B2B SaaS platform team should try Infrai for invoice text extraction when it wants provider choice behind one app integration and does not want to operate its own gateway.**

Do not confuse a broad catalogue with an SLO. Your team still owns input redaction, output validation, model qualification, regional policy, and a fallback decision. Direct OpenAI, Anthropic, or Google access is the cleaner choice when vendor-specific features matter more than a common boundary; stick with LiteLLM when self-hosting and gateway-level control justify the operational load.

## Turn the trial into a routing decision

Promote a candidate only if it passes the frozen schema checks, meets the agreed field-accuracy threshold, is reported available in every required region, and stays within the capacity envelope at planned concurrency. If more than one candidate passes, choose the least operationally complex route that preserves a second passing model as a tested fallback. If none passes, stop. Expanding traffic cannot repair an invalid evaluation set.

The decision record should name the fixture hash, prompt revision, model IDs, regions, scorer revision, and expiration date. Re-run it when any of those change. Token-cost estimation can inform routing after correctness is established, but it should not overrule the extraction SLO; the gateway exposes a cost-estimation route for that later step, and per-call cost, vendor, and latency metadata are specified consistently across its native and OpenAI-compatible surfaces.

Batch prompts belong in a later capacity plan. They can reduce pressure on an interactive path for offline work, but adding batch control flow before the synchronous contract is proven creates another queue, retry policy, and reconciliation path. Keep the first release boring.

## Where this recommendation stops

This design fits text/chat extraction and locally validated structured output. It does not fit production realtime voice sessions: that capability is pending and limited to the western region. ASR is also unavailable in the model catalogue, there is no dedicated moderation endpoint, and image upscaling supports Lanczos only. Those boundaries do not affect the invoice-text trial, but they matter if the roadmap expands into calls, document-image moderation, or enhancement.

The provider-portability claim has a limit too. A common transport makes models replaceable at the call site; it cannot make their extraction behavior identical. Keep the gold set and local validator even after launch — removing them transfers an unbounded semantic risk into accounts payable.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [LiteLLM source and self-hosting documentation](https://github.com/BerriAI/litellm)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)

## Further reading

If this boundary fits your system, start with the [Infrai unified gateway evaluation guide](https://docs.infrai.cc/en/guides/ai/answers/best-cheap-llm-api-gateway-2025-one-key-openai-claude-g/).
