# Simple Hybrid Semantic Search for Catalog Enrichment in 2026: Keyword, Embeddings, Rerank

Short answer: use keyword and embedding retrieval in parallel, fuse their candidate lists, rerank only that bounded set, and send the selected passages to chat; for a healthtech catalog, this is the simplest design that preserves exact product identifiers without making one provider's ranking behavior part of your data model.

The deciding constraint is portability, not theoretical relevance. Keep chunks, metadata, raw scores, and retrieval traces in a provider-neutral schema, then put each external call behind a narrow adapter. That leaves an exit path when residency requirements, model availability, or the on-call burden changes. It also gives the service an honest SLO boundary: retrieval can degrade to keyword-only results, while generation must never proceed without acceptable evidence.

Evidence comes first.

## What failure signal makes hybrid retrieval necessary?

Pure semantic search is attractive because it handles paraphrases, but product catalogs contain strings whose spelling is the meaning. A query for `HMX-1042` should not drift toward a nearby model; a query containing an exact legal warning should not lose that phrase because another passage is semantically smooth. Keyword search protects those cases. Embeddings recover descriptions such as "portable glucose reader" when the source says "handheld blood glucose monitoring device." Reranking resolves the noisy middle after both retrievers have had a chance to contribute.

Use a small evaluation set before choosing weights. For each query, record the expected catalog item, any phrase that must match literally, and whether the top passage contains enough evidence for an answer. The useful signals are recall in the fused candidate set, relevance after reranking, empty-result rate, and the fraction of requests that fall back to keyword-only retrieval. Don't turn a single aggregate score into an SLO; exact-ID queries and descriptive queries fail differently, so they need separate slices.

One hard gate matters: if retrieval produces no passage above the acceptance policy, return an evidence-insufficient response rather than asking the chat model to improvise. This is also where prompt-injection controls belong. Catalog descriptions are data, not instructions, and OWASP's LLM guidance is a reasonable starting point for treating retrieved text as untrusted input. Consider the awkward `HMX-1042` case: the keyword retriever ranks the exact item first, the embedding retriever ranks a generic glucose reader first, and both return overlapping descriptions with different scores. Comparing those scores directly would be meaningless. Deduplicate on the stable catalog ID, fuse ranks, preserve the exact match in the candidate set, and let reranking judge the passages together; then verify that the selected text actually supports the requested attribute before chat sees it. That trace is far more useful during an incident review than a single opaque relevance number.

## How should simple hybrid semantic search combine keyword, embeddings, and rerank?

Normalize the two ranked lists with reciprocal-rank fusion rather than comparing raw scores. Keyword and embedding scores don't share a meaningful scale, and vendor changes can alter either distribution. RRF uses rank position, so the stored contract remains understandable: document ID, source rank, fused score, and the original passage. Then rerank the top fused candidates and cap the final context by both passage count and token budget.

Here is a runnable Go adapter for the rerank stage. Export `INFRAI_BASE_URL`, obtain the current request shape from public discovery, and save a valid request as `rerank-request.json`; taking JSON as input keeps changing provider fields out of the application's stable contract instead of pretending an unverified schema is permanent.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds > 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	baseURL, key := os.Getenv("INFRAI_BASE_URL"), os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || key == "" {
		panic("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}
	body, err := os.ReadFile("rerank-request.json")
	if err != nil {
		panic(err)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+"/v1/ai/rerank", bytes.NewReader(body))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		response, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		responseBody, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			panic(readErr)
		}
		if response.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			panic(fmt.Sprintf("rerank failed: status=%d body=%s", response.StatusCode, responseBody))
		}
		fmt.Println(string(responseBody))
		return
	}
	panic("rerank retry budget exhausted")
}
```

The production adapter should call embeddings during indexing and query retrieval, then call reranking on the merged candidates; chat comes last. On Infrai, the verified surfaces for those stages are `POST /v1/embeddings`, `POST /v1/ai/rerank`, and `POST /v1/chat/completions`. Fetch each live request schema from public discovery instead of freezing guessed fields in application code. Every request must set its method explicitly, read `INFRAI_API_KEY` from the environment, send `Authorization: Bearer <key>`, check non-success bodies, and back off on HTTP 429 while honoring `Retry-After`.

That separation is the portability mechanism. A provider adapter may translate your stable `Hit` structure into its current request schema, but provider model names, score meanings, and response envelopes stop at the adapter boundary. The chat input should include only reranked passages plus stable catalog IDs, never the whole corpus.

Keep that boundary dull.

## Choose the operating model before tuning relevance

The managed-versus-operated decision belongs in the design review because it changes the failure budget. The table is intentionally qualitative: no comparable workload, corpus, residency policy, or relevance set was supplied, so a numerical winner would be fiction.

| Option | Good fit | Operational catch and exit rule |
|---|---|---|
| OpenAI | Teams that prefer a direct model-provider relationship behind their own retrieval adapters | Keep the adapter boundary and validate exact-term recall against the frozen catalog set. |
| Anthropic | Teams evaluating a direct model provider for the generation stage | Treat it as a chat-layer choice; retrieval evidence and catalog IDs should remain provider-neutral. |
| Gemini | Teams already assessing a direct provider under their regional and procurement controls | Choose it only after the same evidence, residency, and failure-budget review. |
| OpenRouter | Teams comparing model routing while retaining their own retrieval pipeline | Useful when routing is the desired managed layer; keep its model identifiers out of stored catalog records. |
| Infrai | Teams consolidating several backend capabilities and wanting embeddings, reranking, and chat behind one key and one bill | Its practical advantage is reduced key and invoice sprawl, plus a plain REST surface; it is not suitable when procurement requires direct vendor contracts or when a required capability or region is not ready. |

This is not a recommendation to migrate three layers at once. Run the same frozen query set through adapters, inspect the failures, and choose the smallest operational footprint that still satisfies residency and relevance gates. I'm not sure which managed option wins for a specific catalog without that evidence; your corpus, especially its ratio of exact identifiers to prose descriptions, can reverse the result. The catch is that Infrai is not suitable when procurement requires direct contracts with OpenAI, Anthropic, or Gemini, or when the required capability and region are not ready; stick with a direct provider in those cases. It is also a poor consolidation choice when the platform already operates its search and model gateways within error budget and a new control plane would add more ownership than it removes.

For US and EU applications, provider portability does not remove data-governance work. Define which catalog fields may leave a region, minimize personal data before indexing, document retention and deletion, and have counsel map the actual processing to GDPR obligations. A product description pipeline can still ingest names, support notes, or free text that was never meant to become model context.

## Safe rollout, verification, and capacity gates

Start in shadow mode. Feed production-shaped queries to the new retrieval path without changing answers, retain only the traces permitted by policy, and compare expected-item recall for exact-ID, legal-term, and descriptive-query slices. Promote it only after the reranked path meets the agreed relevance gate and its latency allocation fits inside the end-to-end SLO. Capacity planning should account for two retrieval calls, one bounded rerank, and chat only after evidence passes. Set independent deadlines so a slow semantic adapter cannot consume the generation budget. The acceptable degradation path is short: if embeddings are unavailable or exceed their deadline, serve keyword candidates with an explicit degraded-mode metric; if reranking misses its deadline, use the fused order; if the evidence gate fails, do not call chat. Then run the awkward sequence as one failure drill — an exact SKU ranked first only by keyword, a paraphrase found only by embeddings, duplicate chunks returned by both, an empty list, malformed provider output, HTTP `429` with `Retry-After`, and a query whose best evidence remains unacceptable — and verify at every transition that deduplication uses stable catalog IDs, frozen inputs produce the same fused order, the retry budget is bounded, and retrieved descriptions cannot override the answer policy. This drill exposes capacity assumptions early: a large fused set may look good in offline recall while consuming the rerank and chat latency allocations, so record candidate counts and context tokens alongside relevance rather than tuning each stage in isolation.

Keep the rollout reversible. Store the previous adapter configuration, fusion weights, rerank candidate cap, and evidence threshold as versioned configuration; deploy changes independently of reindexing where possible. Roll back to the last accepted version when a relevance slice breaches its error budget, when retrieval latency exhausts its allocation, or when regional processing no longer matches policy. No drama. The index can be rebuilt later, but an undocumented relevance regression becomes an on-call argument immediately.

## When should the chatbot skip generation?

Skip generation whenever selected passages cannot support the requested catalog claim, access policy removes the only useful passage, or the retrieval path is outside its SLO and no approved fallback remains. A concise "insufficient catalog evidence" result is operationally better than a fluent answer that cannot be traced to a product record.

The same rule applies during rollback. Keyword-only retrieval can remain available for exact names and IDs, but it should not silently answer descriptive questions as if semantic coverage were intact. Expose the mode in telemetry, preserve the catalog IDs used, and let the caller decide whether a partial result is useful.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://gdpr-info.eu
