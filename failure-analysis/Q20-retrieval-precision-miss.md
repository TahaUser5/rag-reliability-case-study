# Failure Analysis: Q20 — Retrieval Precision Miss

**Bottleneck type:** `RETRIEVAL_PRECISION_BOTTLENECK`  
**Outcome:** System produced an incomplete or incorrect answer because the context window was dominated by redundant content

---

## What Happened

The system was asked a question that required a specific piece of information. Retrieval nominally "succeeded" — the correct information was present in the corpus — but the context window passed to the generator was filled with two near-identical redundant chunks, leaving insufficient room for the actionable content. The generator, working from a context dominated by repeated boilerplate, produced an inadequate answer.

---

## Root Cause

The failure was caused by **exact content duplication in the source document** and an **asymmetry in how the two retrieval paths handle deduplication**:

1. **Duplicate source content.** Auditing the source policy document revealed that an identical (or near-identical) block of text appeared on two separate pages of the PDF. This is a source document authoring artefact.

2. **Dense retrieval indexed both copies.** The dense vector search (Pinecone) indexed each chunk as a separate embedding. Because the text is semantically identical, the dense search correctly returned both vectors as highly relevant — from its perspective, they are two independent pieces of strong evidence for the query. Both were returned and placed in the candidate set.

3. **Sparse retrieval deduplicated correctly.** The BM25 keyword index does implement content-hash deduplication. Log output confirmed: "36 persisted + 36 boot-time → 36 unique" — the sparse path correctly collapsed the duplicate and returned only one copy.

4. **Reranker consumed both dense duplicates.** The Cohere reranker received the merged candidate set (dense + sparse). It assigned the two near-identical dense chunks the top two relevance scores (correctly — they are highly relevant). This consumed 2 of the 3 available `top_n` slots with redundant content. The third slot was assigned to a different chunk that did not contain the target information.

5. **Net effect:** The generator's context window contained zero copies of the actionable chunk — only two copies of the redundant text plus one unrelated chunk.

---

## What This Demonstrates

This failure illustrates several important production RAG considerations:

- **Post-retrieval deduplication is a distinct concern from indexing deduplication.** BM25 dedup at index time is not sufficient if the dense retrieval path bypasses that deduplication. A unified deduplication step *after* merging dense and sparse candidates — before the reranker — would have prevented this failure.

- **Source document quality affects retrieval quality.** Exact content duplication in the source corpus creates adversarial conditions for dense retrieval that are not easily detected upstream. Source document auditing and pre-ingestion deduplication are preventive measures.

- **Reranker slot starvation.** With a fixed `top_n=3`, consuming 2 slots with duplicates leaves only 1 slot for genuinely distinct content. This demonstrates that a precision failure in retrieval can be just as damaging as a recall failure.

- **Diagnosed trade-off:** The fix requires implementing a post-merge, pre-reranker string deduplication step on the dense retrieval branch. This was out of scope for the current baseline but is documented as a concrete improvement path.

---

## Status

**Diagnosed, unpatched.** The root cause is fully understood and confirmed via source document audit. The fix (post-merge content deduplication before reranking) is out of scope for the current evaluation baseline. Documented as a known limitation.
