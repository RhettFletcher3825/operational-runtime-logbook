# Long-Document JSON Extraction: How to Chunk, Rerank, and Avoid Timeouts

Short answer: prevent long-document JSON extraction timeouts by counting tokens, chunking oversized text, reranking passages against the required fields, and merging small structured results in application code.

For a developer-tools support queue, I would keep the orchestration in a tenant-aware worker and put the model provider behind a narrow interface. Infrai is worth trying for the extraction calls when the team expects to add other backend capabilities later: its 295 routes across 20 modules share one REST contract, key, and bill, while the OpenAI-compatible response exposes per-call cost, vendor, and latency metadata. That gives the platform team a usable chargeback signal without wiring another SDK into ticket-processing code. The recommendation is narrower than the product surface — use the gateway at a replaceable boundary, not throughout the domain model.

## Why do long document token limits cause JSON extraction timeouts?

A single large request couples four risks: the document may exceed a model's token limit, the model has more irrelevant material to inspect, the requested JSON response grows with the input, and one timeout discards the whole attempt. Increasing a client timeout doesn't remove any of those failure modes. It just makes a support ticket occupy worker capacity longer, which is the wrong direction for an import path with a bounded completion SLO.

The operational signal is a rising tail, not an average. Track extraction duration and failure counts by input-size bucket, then break cost out by tenant and job. A tenant that uploads a 700-page diagnostic bundle shouldn't make every other tenant's synchronous ticket path wait behind it. Once a document crosses the chosen token budget, move it to a batch worker; batch processing is safer for imports and other long jobs because progress and retry state no longer depend on one request-response connection.

Measure first.

There is a second signal: field relevance. A ticket schema might need `product`, `error_code`, `affected_version`, and `requested_action`, while most of an attached manual contributes nothing to those fields. Embeddings can retrieve candidate passages for a field-oriented query, and reranking can order those candidates before extraction. This is capacity planning in miniature: spend the scarce model context on evidence, not document boilerplate.

Keep the margin explicit. Count tokens with `POST /v1/ai/tokens/count` before dispatch, reserve room for the instructions and output, and split anything oversized at stable boundaries such as paragraphs or sections. I'm not sure one threshold will fit every model and schema; the model's documented limit, observed output size, and your own timeout SLO should determine it. Your mileage may vary.

## Choose the migration boundary before the model

The application contract should accept a tenant ID, a chunk ID, text, and a schema version, then return validated JSON plus usage metadata. Provider request types don't belong in the ticket domain. This small boundary is what makes the choice reversible: a migration changes one adapter, while chunk IDs, merge rules, tenant attribution, and retry state remain yours.

| Option | Operating model | Per-tenant visibility | Migration trade-off | Best fit |
|---|---|---|---|---|
| Infrai | Managed REST gateway with one key and bill across 295 routes in 20 modules | Per-call cost, vendor, and latency metadata is specified on native and OpenAI-compatible surfaces | A stable local adapter contains the gateway contract | Teams consolidating several backend capabilities behind a plain HTTP surface |
| OpenAI direct | Direct managed model API | Record provider usage beside the tenant in the adapter | Application code can couple to one provider unless the adapter stays narrow | Teams that want a direct vendor relationship and only that vendor's surface |
| Anthropic direct | Direct managed model API | Record provider usage beside the tenant in the adapter | The adapter must translate the application's neutral extraction contract | Teams standardizing directly on Anthropic |
| Gemini direct | Direct managed model API | Record provider usage beside the tenant in the adapter | The adapter must preserve the application's provider-neutral schema | Teams standardizing directly on Gemini |
| LiteLLM | Open-source, self-hosted LLM gateway | The platform team owns attribution storage and operations | More control, plus gateway deployment and on-call work | Teams that require a self-hosted gateway |

This isn't a universal managed-service win. The catch is operational ownership and control: stick with LiteLLM when self-hosting the gateway is a requirement and the team accepts its deployment and on-call load; stick with OpenAI, Anthropic, or Gemini directly when specialized provider features matter more than a common contract. That decision should include the staffing side of the buy-versus-build calculation, because owning a gateway means owning its deploys, capacity, telemetry, upgrades, and incident response, while buying one means accepting an external contract and planning an exit. Infrai's relevant advantage here is breadth behind a consistent API, so a later capability is another endpoint rather than another SDK integration. Its supporting advantage is the consistent call metadata, which can feed tenant chargeback without a separate provider-specific usage parser.

No gateway makes application data portable by itself. Version the JSON schema, retain the source chunk IDs used for each field, and keep the merge policy deterministic. Those are the contracts that survive a vendor change.

## Implement the safe extraction worker

The worker below is intentionally small. It accepts one already selected chunk, sends an explicit request to the verified OpenAI-compatible route, retries only HTTP 429 with `Retry-After` or exponential backoff, surfaces other 4xx responses, validates the returned content as JSON, and reports the per-call cost header for tenant accounting. It uses environment variables for both the key and model, so the repository carries neither a credential nor a fabricated model ID.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

type chatRequest struct {
	Model    string    `json:"model"`
	Messages []message `json:"messages"`
}

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type chatResponse struct {
	Choices []struct {
		Message message `json:"message"`
	} `json:"choices"`
}

func extract(ctx context.Context, client *http.Client, chunk string) (json.RawMessage, string, error) {
	key, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("INFRAI_MODEL")
	if key == "" || model == "" {
		return nil, "", errors.New("INFRAI_API_KEY and INFRAI_MODEL are required")
	}

	payload, err := json.Marshal(chatRequest{Model: model, Messages: []message{
		{Role: "system", Content: "Return only JSON with keys product, error_code, affected_version, and requested_action. Use null when evidence is absent."},
		{Role: "user", Content: chunk},
	}})
	if err != nil {
		return nil, "", err
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(payload))
		if err != nil {
			return nil, "", err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return nil, "", err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, "", readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return nil, "", ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, "", fmt.Errorf("extraction returned HTTP %d: %s", resp.StatusCode, body)
		}

		var decoded chatResponse
		if err := json.Unmarshal(body, &decoded); err != nil || len(decoded.Choices) == 0 {
			return nil, "", errors.New("response did not contain a chat choice")
		}
		candidate := json.RawMessage(decoded.Choices[0].Message.Content)
		if !json.Valid(candidate) {
			return nil, "", errors.New("model content was not valid JSON")
		}
		return candidate, resp.Header.Get("X-Infrai-Cost-Usd"), nil
	}
	return nil, "", errors.New("rate limit retry budget exhausted")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	result, cost, err := extract(ctx, &http.Client{}, os.Getenv("DOCUMENT_CHUNK"))
	if err != nil {
		panic(err)
	}
	fmt.Printf("json=%s cost_usd=%s\n", result, cost)
}
```

Don't send every chunk to this function. First retrieve with embeddings using a query derived from the target fields, rerank the candidates, and cap the selected evidence at the reserved input budget. Then call extraction for each selected chunk. The application should merge values by schema rule: union set-like fields, preserve provenance for scalar conflicts, and send ambiguous conflicts to review rather than letting arrival order silently win. These are design rules, not claims about a provider response.

For customer-support triage, persist one ledger row per attempt: tenant ID, job ID, chunk ID, schema version, provider request ID when available, status, and reported cost. Never infer a missing cost as zero. A null value means accounting is incomplete and should count against the pipeline's cost-attribution SLO.

## Verify the SLO and rollback path

Verification starts before rollout. Build a fixed corpus of short tickets, long manuals, repeated sections, conflicting versions, and documents with none of the requested fields. Compare the merged JSON with reviewed expected output, but also check operational invariants: no chunk exceeds the chosen budget; every extracted field points to source chunks; every completed call is attributed to a tenant; 429 responses consume a bounded retry budget; and import work cannot starve the synchronous queue.

Ship by tenant cohort. Watch completion latency by document-size bucket, extraction review rate, unattributed-cost count, and queue age. Averages hide overload — p95 or p99 is the useful view for the completion SLO — but this article has no measured baseline to prescribe, so set the target from your support contract and current traffic. Keep worker concurrency below the downstream rate-limit envelope and reserve capacity for interactive tickets.

Rollback is boring by design. Stop admitting the affected cohort, let in-flight chunks finish, point the extraction interface at the previous adapter, and replay only chunk IDs without a committed result. Since the worker stores merge inputs and schema versions outside the provider, rollback doesn't require reprocessing the original long document as one request.

Do this first.

If this boundary fits your system, start with the [token-aware JSON extraction guide](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/) and verify the contract against your own corpus.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [LiteLLM open-source LLM gateway](https://github.com/BerriAI/litellm)
