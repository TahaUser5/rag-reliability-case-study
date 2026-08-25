# RAG Reliability Case Study
An enterprise-grade Retrieval-Augmented Generation (RAG) HR assistant, evaluated against a 20-question golden dataset with an independent judge model — documenting real failure modes, bottleneck distributions, and the engineering decisions behind a faithfulness score of **0.911**.

📹 **[DEMO VIDEO]**

https://github.com/user-attachments/assets/a4c9d4e0-f147-4553-ac4e-f95650b75ba9

---

## Contents

| Resource | Description |
|---|---|
| [CASE_STUDY.md](./CASE_STUDY.md) | Full narrative: system overview, evaluation methodology, results, and what this demonstrates |
| [methodology.md](./methodology.md) | Evaluation framework design — why an independent judge, how the benchmark was constructed |
| [evaluation/](./evaluation/) | Headline metrics summary and stripped results table |
| [failure-analysis/](./failure-analysis/) | Per-failure root-cause write-ups for Q16, Q17, and Q20 |
| [architecture/](./architecture/) | System architecture diagram — hybrid retrieval + evaluation pipeline |
| [screenshots/](./screenshots/) | UI screenshots of the chat interface and demo assets |
