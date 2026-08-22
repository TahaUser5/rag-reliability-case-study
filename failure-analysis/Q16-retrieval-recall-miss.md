# Failure Analysis: Q16 — Retrieval Recall Miss

**Bottleneck type:** `RETRIEVAL_RECALL_BOTTLENECK`  
**Outcome:** System abstained (returned fallback response) — incorrect, as the answer *was* present in the documents

---

## What Happened

The system was asked a question requiring retrieval of a specific, literal piece of contact information from the policy document. Despite the information being present in the source corpus, the pipeline returned an abstention response — i.e., it said it could not find the answer.

---

## Root Cause

The retrieval failure was caused by an interaction between the **multi-query expansion strategy** and the **reranking step**:

1. **Multi-query expansion misfired.** The retriever was configured to expand a single query into multiple semantically related variants before searching. For this question, the expansion generated variants that were *semantically adjacent but factually unhelpful* — the expanded queries targeted coordinate-style or landmark-style expressions rather than the literal contact block format. This caused the dense vector search to return chunks that were thematically related but did not contain the target information.

2. **The correct chunk was displaced by the reranker.** The footer chunk containing the target information did exist in the candidate set returned by BM25 (keyword search), but the Cohere reranker — evaluating relevance against the expanded query variants — promoted a denser, thematically richer chunk (covering a related topic area) into the top-3 slots, displacing the footer chunk entirely.

3. **No fallback path.** With none of the top-3 reranked chunks containing the answer, the generator correctly abstained rather than hallucinating. This abstention behaviour is *correct pipeline behaviour* — the failure is in retrieval, not generation.

---

## What This Demonstrates

This failure illustrates a well-known trade-off in RAG system design:

- **Query expansion helps breadth, hurts precision on literal lookups.** MultiQuery-style expansion is highly beneficial for broad semantic questions where paraphrasing surfaces relevant content. It is actively harmful for exact, literal string lookups (e.g., phone numbers, addresses, identifiers) where the query should be kept narrow.

- **Reranking amplifies upstream retrieval errors.** If query expansion causes the wrong chunks to be retrieved in the first place, a reranker cannot recover — it can only rank what it was given. The reranker correctly identified the *most relevant* chunk from a bad candidate set.

- **Diagnosed trade-off:** Disabling multi-query expansion for literal lookups would fix this case but hurt broader semantic retrieval. Increasing the reranker `top_n` would slow the pipeline. Both mitigations have system-level costs — this is documented as an architectural trade-off rather than a simple bug.

---

## Status

**Diagnosed, unpatched.** The root cause is fully understood. The fix (conditional query expansion suppression for literal-lookup query types) is out of scope for the current evaluation baseline. Documented as a known limitation.
