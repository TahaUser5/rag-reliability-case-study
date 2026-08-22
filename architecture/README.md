# architecture/

This folder is intended to hold the system architecture diagram for the RAG pipeline.

## What to add manually

- **`architecture-diagram.png`** — Export the architecture diagram from your diagramming tool (Excalidraw, draw.io, Miro, Lucidchart, etc.)

> **Note:** The original architecture diagram is located in the private repository at `docs/architecture.png`. Export or recreate it before adding here. Do not copy the file directly from the private repo if it contains any proprietary annotations.

A good architecture diagram for this system should show:
1. Document ingestion and chunking stage
2. Hybrid retrieval (dense + sparse paths, merging strategy)
3. Reranking stage
4. LLM generation stage
5. Evaluation loop (RAGAS + judge model) as a separate, connected process
