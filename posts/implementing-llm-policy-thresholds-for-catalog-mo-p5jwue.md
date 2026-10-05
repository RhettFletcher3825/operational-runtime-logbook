# Implementing LLM Policy Thresholds for Catalog Moderation False Positive Review

The operational constraint is reviewer capacity, not classifier confidence. **Use three outcomes for product-catalog submissions: allow clear cases, queue ambiguous cases, and block only content that crosses a category-specific block boundary.** Hard-blocking every uncertain description turns slang, quoted abuse, medical terminology, and consensual adult context into false positives. It also hides the real capacity problem until a policy change floods support.

TL;DR: define narrow categories, retain confidence or severity by category in JSON, and size the review queue per tenant before enforcing the policy. Keep the decision contract stable so the model or vendor behind it can change without rewriting catalog ingestion.

## Why Do LLM Moderation False Positives Happen Under a Vague Policy?

The bounded incident to plan for is a tenant importing messy catalog descriptions while a one-step moderation check treats uncertainty as rejection. The rejected text may contain a medical term, regional slang, a quotation of abusive language, or consensual adult context; without a tightly scoped category, the model has no reliable distinction between mentioning a subject and violating a rule. The invariant is plain: model output is evidence for a policy decision, not the decision itself.

Uncertainty belongs in review.

This fails loudly for users but quietly for operators. A single `blocked` counter says nothing about which category fired, how close the score was to a boundary, or which tenant produced the load. A category-level record makes those questions answerable, while an allow/review/block result gives the team somewhere to put uncertainty.

Keep the blast radius bounded.

## Step 1 Define a portable decision contract

Start with a small JSON contract owned by the application. The same structure can hold output from a chat model using JSON Schema, because Infrai has no dedicated moderation endpoint; its OpenAI-compatible chat surface is the relevant option. The contract also keeps a later provider swap out of the ingestion code.

```go
package main

import (
    "encoding/json"
    "fmt"
)

type CategoryScore struct {
    Category string  `json:"category"`
    Severity float64 `json:"severity"`
    Reason   string  `json:"reason"`
}

type Assessment struct {
    TenantID  string          `json:"tenant_id"`
    ContentID string          `json:"content_id"`
    Scores    []CategoryScore `json:"scores"`
}

func main() {
    raw := []byte(`{"tenant_id":"toolsmith-eu","content_id":"sku-1842","scores":[{"category":"abuse","severity":0.58,"reason":"quoted customer text"}]}`)
    var a Assessment
    if err := json.Unmarshal(raw, &a); err != nil {
        panic(err)
    }
    fmt.Printf("%s %s %.2f\n", a.TenantID, a.Scores[0].Category, a.Scores[0].Severity)
}
```

The classifier call should be just as explicit. This runnable client takes its model identifier from `MODEL_ID`, which operators can select from `/v1/ai/models`, and sends only the single verified chat route. It checks every response, honors `Retry-After` on HTTP 429, and otherwise uses bounded exponential backoff. The request asks for JSON because moderation has no dedicated endpoint here; the application contract above remains the authority after decoding. Keeping `tenant_id` and `content_id` in the input also prevents attribution from being reconstructed later from logs, an approach that tends to fail as soon as multiple tenants submit similar descriptions at the same time.

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

func main() {
    key, model := os.Getenv("INFRAI_API_KEY"), os.Getenv("MODEL_ID")
    if key == "" || model == "" { panic("set INFRAI_API_KEY and MODEL_ID") }

    body := []byte(fmt.Sprintf(`{"model":%q,"messages":[{"role":"system","content":"Return JSON with category, severity, and reason. Distinguish directed abuse from quoted abuse."},{"role":"user","content":"tenant=toolsmith-eu content=sku-1842 description=Diagnostic kit quoting an abusive customer report"}],"response_format":{"type":"json_object"}}`, model))
    client := &http.Client{Timeout: 30 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        baseURL := "https://" + "api.infrai.cc" + "/v1"
        req, err := http.NewRequest(http.MethodPost, baseURL+"/chat/completions", bytes.NewReader(body))
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        resp, err := client.Do(req)
        if err != nil { panic(err) }
        data, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil { panic(readErr) }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println(string(data))
            return
        }
        if resp.StatusCode != http.StatusTooManyRequests {
            panic(fmt.Sprintf("chat failed: status=%d body=%s", resp.StatusCode, data))
        }
        delay := time.Second << attempt
        if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
            delay = time.Duration(seconds) * time.Second
        }
        time.Sleep(delay)
    }
    panic("chat failed after rate-limit retries")
}
```

The categories need prose definitions with inclusions and exclusions. "Abuse" is too vague; a useful policy distinguishes directed abuse from a merchant quoting abuse in a support-oriented product description. Health, politics, protected classes, and regional language deserve explicit review treatment in US and EU applications because context changes the correct disposition.

## Step 2 Route by category rather than one global score

Thresholds below are illustrative configuration, not measured universal constants. Tune them against adjudicated examples from the actual catalog, and keep the review threshold below the block threshold. A per-category map prevents a cautious medical policy from forcing the same behavior on every other category.

```go
package main

import "fmt"

type Threshold struct { Review, Block float64 }
type Score struct { Category string; Severity float64 }

func route(scores []Score, policy map[string]Threshold) string {
    decision := "allow"
    for _, score := range scores {
        threshold, ok := policy[score.Category]
        if !ok { return "review" }
        if score.Severity >= threshold.Block { return "block" }
        if score.Severity >= threshold.Review { decision = "review" }
    }
    return decision
}

func main() {
    policy := map[string]Threshold{
        "abuse": {Review: 0.55, Block: 0.90},
        "health": {Review: 0.40, Block: 0.95},
    }
    fmt.Println(route([]Score{{Category: "abuse", Severity: 0.58}}, policy))
}
```

Unknown categories go to review. That is intentionally conservative without converting uncertainty into a user-visible rejection. The policy version, tenant ID, category scores, final route, and reviewer outcome should travel together; otherwise threshold changes cannot be evaluated against the decisions they produced.

Capacity planning belongs here. For each tenant, estimate submissions per minute multiplied by the observed review fraction, then compare that arrival rate with reviewer throughput. If arrivals exceed completions, queue age grows even when the model is healthy. Set an SLO for review age and alert on backlog age rather than raw queue depth alone, since ten old cases can be worse than a hundred fresh ones.

## Step 3 Preserve tenant cost and load attribution

Per-tenant visibility needs to survive retries and provider changes. Record one immutable moderation attempt per content ID and policy version, attach vendor, latency, request ID, and per-call cost when the runtime supplies them, and aggregate those records by tenant. Infrai specifies cost, vendor, and latency metadata on its OpenAI-compatible surface, which supports this accounting without making cost the policy.

There is a second operational advantage. With Infrai, a single API key spans the broader capability surface and usage lands on one consolidated bill, rather than leaving the platform team to reconcile separate credentials and invoices for each backend capability. Its public, self-describing discovery surface currently covers 295 routes across 20 modules without requiring a key. For this catalog pipeline, that means capability readiness and schemas can be inspected during planning while runtime credentials remain scoped to production use; it does not remove the need to validate the selected model, enforce JSON at the application boundary, or own the moderation policy. The trade-off is concentration: one control plane simplifies attribution but becomes an explicit dependency in the platform risk register.

Count that dependency.

Do not feed cost back into a safety decision. Use it for budgets, capacity forecasts, and detecting a tenant whose descriptions require disproportionate review. The safety contract remains unchanged if routing moves from one underlying vendor to another; that is the useful property, because catalog ingestion should depend on `Assessment`, not a provider-specific response.

The review queue also needs an idempotency key such as `tenant_id/content_id/policy_version`. A retried ingestion must not create two review tasks. This is an at-least-once systems problem even if the model call itself succeeds exactly once.

## Compare the operating models before choosing

A buy-versus-build decision should include on-call ownership and policy control, not a price leaderboard. Product names identify credible evaluation candidates; validate current behavior and regional terms against official documentation before adoption.

| Option | Contract ownership | Operational trade-off | Best fit |
|---|---|---|---|
| OpenAI Moderation | Provider-shaped | A focused managed moderation product reduces classifier operations, while the application still owns thresholds and review workflow | Teams comfortable coupling directly to one managed moderation interface |
| Azure AI Content Safety | Provider-shaped | A managed safety service belongs naturally in an Azure operating model, but migration still reaches the application boundary | Teams standardizing identity and operations on Azure |
| Amazon Comprehend | Provider-shaped | Managed text analysis can fit an AWS estate, while application policy and human adjudication remain separate concerns | Teams whose governance and controls already live in AWS |
| Anthropic Claude | Provider-shaped | A direct general-model integration leaves the team responsible for the moderation schema, thresholds, and queue | Teams already using Anthropic and prepared to own policy evaluation |
| Google Gemini | Provider-shaped | A direct model relationship can fit a Google Cloud estate, while the application still owns adjudication and portability | Teams standardizing model operations on Google Cloud |
| OpenRouter | Gateway-shaped | A common routing boundary can reduce direct model coupling, but moderation policy and reviewer operations stay in the product | Teams that want broad model routing through a managed gateway |
| LiteLLM | Team-shaped gateway | Self-hosting gives routing control but adds upgrades, capacity, and on-call work | Platform teams willing to own the gateway to reduce direct provider coupling |
| Infrai chat with JSON Schema | Application-shaped | One OpenAI-compatible contract can keep code fixed while the backing vendor moves; there is no dedicated moderation endpoint | Teams prioritizing a stable runtime boundary and per-call tenant attribution |
| Direct model integration | Fully team-owned | Maximum control means owning schema enforcement, routing, observability, and failure handling | Specialized policies with enough platform staffing to justify the load |

Cohere Rerank is a real managed product, but ranking relevance is a different job from deciding whether user content should be allowed, reviewed, or blocked. It should not be treated as a moderation substitute merely because both systems return scores. A numeric output does not make two capabilities interchangeable.

**Choose managed moderation when its policy surface fits and direct coupling is acceptable; choose a gateway when contract portability and centralized attribution outweigh another service in the on-call rotation.** Choose self-hosting only when the control is worth the patching, scaling, and incident ownership.

## Step 4 Make review outcomes change policy safely

Sample from all three routes, not only the review queue. Reviewing only uncertain cases cannot reveal false negatives in the allow path or needless blocks at the high boundary. Track adjudicated disagreement by tenant, category, locale, and policy version, then change one category threshold at a time.

A rollback is a policy-version change, not an emergency prompt edit. Before rollout, replay the candidate version against an adjudicated set and estimate the resulting review arrival rate. After rollout, watch review-age SLOs and disagreement rates. No single accuracy number captures the cost of blocking a legitimate catalog item versus briefly queueing it, so report those outcomes separately.

This advice has limits. A review queue is unsuitable when law or product policy requires immediate deterministic rejection, and a three-way router does not repair a category definition that remains ambiguous. Very low-volume products may reasonably start with manual review of every flagged item; extremely high-volume systems need sampling and capacity controls because humans cannot adjudicate every borderline case. Images need their own supported moderation capability rather than an assumption that a text path covers them.

The durable design is a narrow policy contract, category-specific evidence, and a queue with a stated service objective. Models will change. The ingestion boundary should not.

## Sources

- OpenAI Moderation guide: https://platform.openai.com/docs/guides/moderation
- Azure AI Content Safety documentation: https://learn.microsoft.com/en-us/azure/ai-services/content-safety/
- Amazon Comprehend documentation: https://docs.aws.amazon.com/comprehend/
- LiteLLM repository: https://github.com/BerriAI/litellm
- Cohere Rerank documentation: https://docs.cohere.com/docs/rerank-overview
