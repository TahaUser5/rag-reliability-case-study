# Failure Analysis: Q17 — Generator Miss

**Bottleneck type:** `GENERATOR_BOTTLENECK`  
**Outcome:** System produced an incomplete answer despite having retrieved the correct, complete context

---

## What Happened

Retrieval was successful for this question. The correct chunk was ranked first by the reranker and placed as the primary context in the generator's prompt. The generator nevertheless produced an incomplete answer — correctly quoting one portion of the context but failing to incorporate a second, distinct piece of information that was also present in the same chunk.

---

## Root Cause

This is a **pure generation-stage failure** with no retrieval component:

1. **Retrieval was perfect.** Context precision and context recall were both high. The top-ranked chunk contained all the information needed to construct a complete, correct answer. This was confirmed by the RAGAS metrics and by manual inspection of the bottleneck classification.

2. **The LLM produced a partial synthesis.** The generator model (`llama-3.1-8b-instant`) extracted one piece of information from the chunk and returned it as the complete answer. A second, distinct factual element present in the same passage was omitted.

3. **Likely cause: over-literal extraction.** The chunk contained information in a form that made one element salient (e.g., appearing first or in a simple sentence) while a second element was embedded in a more complex clause. The model appears to have satisfied its "found an answer" stopping condition on the first element without continuing to synthesise the second.

4. **Not a hallucination.** The answer produced was grounded — it contained no information that wasn't in the context. Faithfulness score was high. This is a **completeness failure**, not a factual accuracy failure.

---

## What This Demonstrates

This failure illustrates an important distinction in RAG failure modes:

- **Retrieval success ≠ answer success.** A high-quality RAG system must succeed at all three stages independently: chunking/ingestion, retrieval, and generation. This case demonstrates that even with a perfect top-3 context set, the generation model can still underperform.

- **Completeness is harder to measure than faithfulness.** RAGAS faithfulness measures grounding (are all claims supported?). Answer completeness — did the model cover *everything* it should have? — is a distinct, harder-to-measure property. The answer relevancy metric partially captures this, but is less sensitive for partial-answer failures on multi-element questions.

- **Model capability trade-off.** Using a smaller, faster generator model (`llama-3.1-8b-instant`) trades inference cost for generation quality. A larger generator model would likely synthesise multi-element answers more reliably. This is a documented design trade-off, not a pipeline bug.

- **Mitigation paths:** Chain-of-thought prompting, explicit "list all relevant facts" instructions, or a post-generation answer completeness check against retrieved context are all viable mitigations — each with their own latency and cost trade-offs.

---

## Status

**Diagnosed, unpatched.** Root cause is fully understood. The failure is attributed to the generator model's tendency to under-synthesise multi-element answers from long chunks. Documented as a known limitation of the current generator model choice at the current evaluation baseline.
