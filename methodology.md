# Evaluation Framework — Methodology

## Overview

The RAG pipeline was evaluated against a manually curated 20-question golden dataset using the **RAGAS evaluation framework (v0.4.3)**. This document describes the design decisions behind the evaluation methodology.

---

## Benchmark Design

### Dataset Construction

The golden dataset was hand-authored to cover a range of question difficulty and retrieval challenge:

- **Straightforward factual lookups** — Questions with a clear, retrievable answer in the source documents
- **Multi-chunk synthesis questions** — Questions requiring information from more than one section
- **Literal string lookups** — Questions targeting specific contact details, numbers, or addresses, which stress-test keyword retrieval
- **Unanswerable queries** — At least one question explicitly designed to have no answer in the provided documents, testing the system's abstention behaviour

Each question in the dataset is paired with a ground-truth reference answer, enabling automated metric computation.

### Benchmark Size

The benchmark consists of **20 questions**, producing an **80-job RAGAS evaluation run** (20 questions × 4 core metrics computed per question). The full run completed in approximately 1 hour 47 minutes, reflecting real-world API rate constraints encountered during evaluation (see below).

---

## Why an Independent Judge Model?

A fundamental design principle of this evaluation is that **the model which generates answers must not be the same model that grades them**.

### The Self-Grading Bias Problem

When the same model both generates and evaluates answers, it introduces systematic bias:

- The model tends to award high faithfulness scores to its own paraphrases, because it recognises its own phrasing style as "grounded"
- Self-generated answers that hallucinate are harder for the same model to detect as hallucinations, since it may have high internal confidence in the (incorrect) claim
- The evaluation becomes circular — the model is marking its own homework

This was an actual failure identified during development: an early version of the evaluation script used `llama-3.1-8b-instant` as both the generator and the judge. This was corrected before the final baseline run.

### The Fix: Independent Judge

The final evaluation uses:

| Role | Model |
|---|---|
| **Generator (student)** | `llama-3.1-8b-instant` |
| **Judge (evaluator)** | `openai/gpt-oss-120b` (via Groq) |

The judge is a distinctly higher-capability model with no relationship to the generator, ensuring objective, third-party scoring of faithfulness and relevancy claims.

---

## Metrics Computed

| Metric | What It Measures |
|---|---|
| **Faithfulness** | Does the answer contain only claims that are directly supported by the retrieved context? (Hallucination resistance) |
| **Answer Relevancy** | Is the answer on-topic and relevant to the question asked? |
| **Context Precision** | Are the retrieved chunks actually useful for answering the question? (Retrieval quality — signal vs. noise) |
| **Context Recall** | Did the retrieval surface all the information needed to construct a correct answer? |
| **Context Recall (Answerable)** | Context recall restricted to non-abstention questions (excludes intentional abstentions from the denominator) |

---

## Bottleneck Classification

After RAGAS scoring, each question was classified into one of five bottleneck categories to identify *where* in the pipeline failures occurred:

| Bottleneck | Meaning |
|---|---|
| `NO_BOTTLENECK` | Pipeline passed — both retrieval and generation succeeded |
| `RETRIEVAL_RECALL_BOTTLENECK` | The correct information was not retrieved at all |
| `RETRIEVAL_PRECISION_BOTTLENECK` | Retrieval returned content, but it was noisy or redundant, crowding out the useful chunk |
| `GENERATOR_BOTTLENECK` | Correct context was retrieved, but the LLM failed to synthesise a complete or accurate answer |
| `AMBIGUOUS_BOTTLENECK` | Multiple simultaneous failure modes, not clearly attributable to one stage |

---

## Rate-Limit Constraints

Running a 20-question evaluation with an independent high-capability judge model on free-tier API access required engineering around real-world constraints:

- **Groq token-per-minute ceilings** — The judge model's throughput required exponential backoff on API calls
- **Cohere trial key rate limit** (10 requests/minute) — Mandatory request pacing was implemented to prevent mid-run crashes

These constraints are documented here because they directly affected the design of the evaluation script and the total wall-clock time of the run (~1h47m for 80 jobs). A production deployment with paid API access would remove these constraints entirely.

---

## Known Metric Artefacts

- **NaN Faithfulness (2 questions):** Two questions returned `NaN` faithfulness due to an HTTP 400 `json_validate_failed` error from the judge model. The judge failed to emit a valid JSON claim-decomposition schema when confronted with unusually long, multi-bulleted answers in a single inference attempt. This is a **metric-infrastructure failure**, not a system failure. The mean faithfulness of **0.911** is computed over the 18 questions where the metric was successfully produced.

---

*No pipeline source code, configuration files, API keys, or raw document content is included in this repository.*
