# Case Study: Building a Reliable RAG HR Assistant

## System Overview

This project is an enterprise-grade Retrieval-Augmented Generation (RAG) knowledge base designed to function as an HR policy assistant. The system ingests internal company policy documents and answers employee questions by retrieving the most relevant information before generating a response. It is designed to be highly factual, resistant to hallucinations, and capable of correctly abstaining when an answer is not present in the provided documents.

The pipeline architecture centres on three core stages:

1. **Ingestion & Chunking** — Documents are loaded and segmented using a semantic overlap strategy. A content-density filter actively rejects low-signal chunks (e.g., table-of-contents entries and structural boilerplate) that would otherwise pollute retrieval.

2. **Hybrid Retrieval** — A two-pronged retrieval strategy merges results from dense vector search (semantic similarity via embeddings) and sparse keyword search (BM25). The merged candidate set is then re-ranked by a neural reranker, returning the top 3 most relevant context chunks.

3. **Grounded Generation** — An LLM generates the final answer strictly from the retrieved context, with a fallback abstention response for queries where no relevant policy is found.

---

## Evaluation Methodology

The system was evaluated against a curated 20-question golden dataset covering a range of HR policy topics — including answerable procedural questions, factual lookups, and a deliberately unanswerable query to test abstention behaviour.

Evaluation was powered by the **RAGAS framework** (v0.4.3), measuring five metrics per question:

- **Faithfulness** — Does the answer contain only claims supported by the retrieved context?
- **Answer Relevancy** — Is the answer relevant to the question asked?
- **Context Precision** — Are the retrieved chunks actually useful for answering?
- **Context Recall** — Did retrieval surface all the information needed?
- **Context Recall (Answerable)** — Context recall restricted to non-abstention questions.

The full evaluation run comprised **80 RAGAS jobs** and completed in approximately 1 hour 47 minutes.

### Independent Judge Model

A critical methodological decision was the use of a **separate, high-capability judge model** to score answers rather than using the same model that generated them. Self-grading introduces systematic bias — a model is more likely to award itself high faithfulness scores by treating its own paraphrases as faithful to the source. Using an independent judge enforces objective, third-party scoring.

> **Judge:** `openai/gpt-oss-120b` (accessed via Groq)  
> **Generator:** `llama-3.1-8b-instant` (accessed via Groq)

---

## Results

### Headline Metrics

| Metric | Score |
|---|---|
| **Mean Faithfulness** | **0.911** *(18 computable questions)* |
| Answer Relevancy | 0.660 |
| Context Precision | 0.821 |
| Context Recall | 0.850 |
| Context Recall (Answerable) | 0.895 |
| Abstention Rate | 10% (2 / 20 questions) |

> **Note on NaN Faithfulness (Q10 & Q11):** Two questions returned `NaN` faithfulness. Post-hoc log analysis confirmed this was caused by an HTTP 400 `json_validate_failed` error from the judge model when forced to decompose unusually long, multi-bulleted answers into a structured JSON claim schema in a single attempt. This is a **metric-infrastructure failure**, not a hallucination event. The mean of 0.911 is computed across the 18 questions where the metric was computable.

### Bottleneck Distribution

| Classification | Count | Notes |
|---|---|---|
| **NO_BOTTLENECK** | **17** | Pipeline passed: correct retrieval + correct generation |
| RETRIEVAL_RECALL_BOTTLENECK | 1 | Q16 — failed to retrieve the correct chunk |
| RETRIEVAL_PRECISION_BOTTLENECK | 1 | Q20 — context window flooded with redundant chunks |
| GENERATOR_BOTTLENECK | 1 | Q17 — correct context retrieved, incomplete synthesis |

> **Classifier Correction:** The automated pipeline classifier counts 16 NO_BOTTLENECK. The corrected count is 17, because Q01 (cell phone policy) is a correct, intentional abstention to a query explicitly marked `UNANSWERABLE` in the golden dataset. The classifier misidentifies this correct abstention as a recall miss. The three genuine failures are Q16, Q17, and Q20.

---

## Failure Analysis

Three genuine failures were identified and diagnosed. See the [failure-analysis/](./failure-analysis/) folder for per-failure root-cause write-ups.

### Q16 — Retrieval Recall Miss
The system failed to retrieve the correct contact/address chunk. The retriever's multi-query expansion strategy generated semantically adjacent but ultimately unhelpful query variants (coordinates, landmarks), causing the dense retrieval path to miss the footer chunk where the relevant information resided. A denser, higher-priority chunk was then promoted by the reranker, displacing any chance of recovery.

### Q20 — Retrieval Precision Miss
The context window for this question was consumed by two near-identical chunks, leaving insufficient space for the actionable content. The root cause is exact duplicate text across two pages of the source document; the dense retrieval path indexed these as separate vectors and returned both, consuming two of the three reranker slots with redundant content.

### Q17 — Generator Miss
Retrieval was perfect for this question — the correct chunk was ranked first. The failure occurred entirely in the generation stage: the LLM synthesised a partial answer, correctly quoting one portion of the context but failing to incorporate a second, distinct piece of information present in the same chunk.

---

## What This Demonstrates

This project demonstrates the full production engineering lifecycle of a RAG system — not just a happy-path demo:

- **Identifying and fixing evaluation bias** (switching from self-grading to independent judge)
- **Engineering robust retrieval** (hybrid BM25 + dense search, content-density filtering, reranking)
- **Systematic failure diagnosis** (bottleneck classification, per-question log analysis)
- **Reliability engineering for API constraints** (exponential backoff, request pacing for rate-limited services)
- **Honest reporting** — distinguishing metric-infrastructure failures from genuine system failures, and documenting diagnosed-but-unresolved trade-offs

---

## Known Trade-offs and Unresolved Items

| Issue | Status | Rationale for leaving unresolved |
|---|---|---|
| NaN Faithfulness (Q10/Q11) | Diagnosed, unpatched | Fixing requires breaking open internal RAGAS metric calls to inject custom retry logic — out of scope |
| Duplicate vector retrieval (Q20) | Diagnosed, unpatched | Fix requires post-retrieval string deduplication on the dense retrieval branch before merging |
| MultiQuery expansion missing explicit facts (Q16) | Diagnosed, unpatched | Disabling MultiQuery hurts broader semantic queries; increasing top_n slows the pipeline |

---

## Future Scope

- **Multi-tenant scaling** — Moving to a production sparse index and adding metadata filtering for tenant isolation
- **Production traffic** — Upgrading from free-tier API keys to remove rate-limit constraints
- **Agentic extension** — Wrapping the RAG chain in an agent framework to support user-specific state queries (e.g., remaining PTO balance) beyond static policy lookup

---

## Services This Demonstrates

This case study is representative of consulting and freelance engagements including:

- RAG pipeline design and evaluation
- Retrieval architecture (hybrid search, reranking)
- LLM evaluation framework implementation
- Production-readiness auditing for AI systems
- Honest failure analysis and trade-off documentation

---

*Source: Internal project retrospective. No proprietary document content, raw prompts, API keys, or pipeline source code is included in this repository.*
