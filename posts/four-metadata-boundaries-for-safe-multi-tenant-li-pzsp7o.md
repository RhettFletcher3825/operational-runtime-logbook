# Four Metadata Boundaries for Safe Multi-Tenant Listing Search

Use metadata filters as an authorization boundary applied before similarity ranking, then use the same metadata to control freshness and retrieval scope. For a B2B SaaS help center that aggregates listings from several sources, every chunk should carry at least a tenant identifier, source identity, parent-document identity, and lifecycle state; the query must constrain those fields before any candidate can reach generation. Semantic similarity answers "what looks relevant?" It does not answer "what may this caller see?"

This distinction matters because retrieval-augmented generation combines retrieved material with a generator. A semantically excellent result from the wrong tenant is still a data disclosure, while a deleted or superseded listing from the right tenant can produce a confident but stale answer. **Filtering defines the eligible corpus; vector ranking only orders that corpus.**

## What do metadata filters protect in multi-tenant vector search?

A vector index stores representations that make approximate similarity search practical. Metadata is the structured context beside each vector: values such as `tenant_id`, `source_id`, `document_id`, `revision`, `published_at`, and `status`. A filter is a predicate over that context, evaluated as part of retrieval, that removes ineligible records from consideration.

The first boundary is tenancy. A request authenticated for tenant `acme` should search only chunks whose `tenant_id` is `acme`; the tenant value must come from trusted request context, never from an unconstrained model-generated query or a client field accepted without authorization. The second boundary is source scope, useful when a help-center answer should cover the public catalog but exclude a tenant's draft import. The third is document state: active, superseded, or deleted. The fourth is time or revision, which prevents an older copy from surviving beside a replacement and winning on semantic score.

These fields do different jobs. Combining them into one opaque label makes migrations and incident analysis harder, because an operator cannot tell whether a miss came from tenancy, source selection, publication state, or revision handling.

Scope first.

## Treat ingestion and retrieval as one control loop

Chunking determines what can be retrieved; metadata determines where and when that chunk is eligible. For aggregated listings, preserve the listing as the parent document and split only content that exceeds the retrieval unit you can reasonably send downstream. Each child needs the same security and lifecycle metadata plus a stable parent identifier. Otherwise, one child can outlive a deletion or move to the wrong tenant during a partial reindex.

Freshness is the awkward part. Multiple sources may publish the same listing on different schedules, and an update can change descriptive text without changing the upstream identifier. Consider a listing split into four chunks: if an importer overwrites chunks one and two before failing, similarity search can assemble an answer from two new fragments and two old ones even though every record belongs to the correct tenant. A timestamp filter alone does not prove that the revision is complete. Use an immutable revision or content digest to distinguish versions, and make activation explicit: write all chunks for the new revision, verify the expected set against the parent, mark that revision active, then retire the previous revision. The trade-off is temporary duplicate storage and a more deliberate ingest state machine in return for an atomic read boundary that operators can inspect and roll back. Do not expose half an update.

Stale is wrong.

The safe order is narrow:

1. Authenticate the caller and resolve the tenant server-side.
2. Build a filter from trusted tenant, allowed sources, and active lifecycle state.
3. Retrieve a bounded candidate set inside that scope.
4. Join or group chunks by parent listing before constructing model context.
5. Record the applied scope and revision identifiers for audit and debugging, without logging private chunk text by default.

Four boundaries are enough to reason about the design. They are not a universal schema. A regulated deployment may need region, retention class, or per-document access controls, but each added dimension increases filter cardinality, test combinations, and index-planning work.

## Implement a fail-closed query contract

The retrieval interface should make an unscoped search impossible to express. The following Go example keeps authorization outside the natural-language query and requires a tenant plus explicit lifecycle state. It deliberately omits database-specific syntax; the adapter must translate the typed filter into the engine's native predicate without dropping fields.

```go
package retrieval

import (
	"context"
	"errors"
	"time"
)

type Scope struct {
	TenantID      string
	AllowedSource []string
	AsOf          time.Time
}

type Filter struct {
	TenantID      string
	SourceIDs     []string
	Status        string
	PublishedBy   time.Time
}

type Hit struct {
	ChunkID   string
	DocumentID string
	Revision  string
	Score     float32
	Text      string
}

type Index interface {
	Search(ctx context.Context, embedding []float32, filter Filter, limit int) ([]Hit, error)
}

func Retrieve(ctx context.Context, index Index, vector []float32, scope Scope, limit int) ([]Hit, error) {
	if scope.TenantID == "" {
		return nil, errors.New("tenant scope is required")
	}
	if scope.AsOf.IsZero() {
		return nil, errors.New("freshness boundary is required")
	}
	if limit < 1 || limit > 50 {
		return nil, errors.New("limit must be between 1 and 50")
	}

	filter := Filter{
		TenantID:    scope.TenantID,
		SourceIDs:   scope.AllowedSource,
		Status:      "active",
		PublishedBy: scope.AsOf,
	}
	return index.Search(ctx, vector, filter, limit)
}
```

The adapter is part of the security boundary. Test the generated predicate, not merely the `Filter` value passed into a mock, because a serialization bug that omits `TenantID` leaves application tests green while widening production retrieval. Also decide what an empty allowed-source list means. A reasonable contract is "no additional source restriction" only when tenant scope remains mandatory; teams that use source membership for authorization should fail closed instead.

Make that contract boring.

Never repair scope after retrieval by discarding unauthorized hits in application code. Post-filtering can return too few useful results, consumes capacity on candidates that should never have been searched, and creates more places where logs, traces, caches, or rerankers can observe the wrong tenant's data.

## Capacity planning changes with filter selectivity

Metadata filters are operational controls, but their shapes affect latency and recall. An engine may apply a predicate before, during, or after approximate nearest-neighbor traversal; the observable question is whether selective filters still satisfy the search SLO at realistic tenant sizes. Measure it. Documentation alone cannot establish the behavior of your data distribution.

| Decision | Managed service | Self-hosted index | SRE question |
|---|---|---|---|
| Filter execution | Implementation and tuning are largely provider-controlled | Team owns index layout and tuning | Does p99 hold for the smallest and largest tenants? |
| Isolation model | Shared namespaces or separate indexes may be available | Either pattern adds operational work | What is the blast radius of a bad predicate? |
| Freshness | Ingestion visibility semantics vary by system | Team owns queues, compaction, and activation | How is read-after-update verified? |
| On-call load | Less infrastructure ownership, more dependency risk | More control, more paging surface | Who diagnoses recall loss at 03:00? |
| Portability | Native filter syntax can create lock-in | Generic contracts still require adapters | Can conformance tests run against a replacement? |

There is no honest universal winner in that table. A separate index per tenant offers a strong operational boundary but multiplies index lifecycle work; a shared index is easier to consolidate but makes the mandatory tenant predicate critical. Cost belongs in the capacity model, alongside query rate, vector count, update churn, and on-call load, rather than serving as the primary argument.

Set an SLO around the user-visible retrieval path, then segment it by filter selectivity and tenant size. A global p99 can hide one large tenant monopolizing capacity or many small tenants paying a fixed search overhead. Track empty-result rate, filtered candidate count where available, active-revision lag, and duplicate parent listings. Keep labels bounded: raw tenant identifiers in metrics can create a cardinality problem, so detailed tenant evidence is usually better placed in sampled, access-controlled logs.

## Verify isolation, freshness, and rollback

Start with adversarial fixtures. Create two tenants whose listings use nearly identical wording, embed both, query as one tenant, and assert that every returned hit has the authorized tenant identifier. Repeat with an active and superseded revision of the same listing. Then delete the parent and verify that no child chunk remains retrievable. These tests should run against the real adapter and an actual index in CI or a pre-production environment, since an in-memory fake cannot prove predicate translation.

For deployment, shadow the new filter or schema against recorded, sanitized queries before making it authoritative. Compare eligible document IDs, not just top scores. A zero-result spike may mean the filter is correct and prior behavior was unsafe, so rollback criteria need two dimensions: isolation must never regress, while availability and freshness can trigger a controlled rollback to the previous known-safe schema or index snapshot.

Rollback the index pointer, not the authorization rule. Keep the previous complete revision addressable until the new revision passes count, tenant, and lifecycle checks; if activation fails, direct reads back to that complete revision. Never restore service by removing `tenant_id = authorized_tenant`.

The release gate should include cross-tenant negative tests, revision completeness, deletion propagation, latency under selective predicates, and a forced adapter failure. **A missing or malformed security field must produce an error, not a broad search.** The generator should receive no context when retrieval fails closed, and the user-facing path should return a bounded failure rather than improvising from model memory.

## Operating rule

Metadata filtering in multi-tenant vector search is structured eligibility, not relevance tuning. Bind tenant scope from authenticated context, carry it on every chunk, combine it with explicit source and lifecycle fields, and enforce the predicate inside retrieval. Then test the adapter with hostile near-duplicates and operate freshness as an atomic revision change.

That rule is deliberately conservative. It accepts some schema work, conformance testing, and possible recall trade-offs because the alternative expands the disclosure boundary and makes stale listing behavior difficult to explain. Semantic ranking begins only after authorization and freshness have narrowed the corpus.

## References

- https://arxiv.org/abs/2005.11401
