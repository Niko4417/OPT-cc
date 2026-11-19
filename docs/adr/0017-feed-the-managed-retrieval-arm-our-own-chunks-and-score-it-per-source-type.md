---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/41
---

# Feed the managed retrieval arm our own chunks and score it per source type

The app pushes the same tree-sitter and Docling chunks the self-built arms use into the Bedrock Managed Knowledge Base through its custom connector, one chunk per record, with chunking set to none. The bake-off then measures the managed retrieval, not the managed chunker, which treats code as plain text. The knowledge base still parses each record and embeds it with its own model, which cannot be changed. Audio and video go in as transcripts, never as raw files, because storage is billed per GB of raw data. Results are scored per source type (code and prose) and per question type.

Answers to the standing architecture questions for this arm:

- **Retrieval per source type.** Code and prose both arrive as our own chunks. The arm's results are scored separately for code and prose.
- **Scale.** One knowledge base per project memory. Storage costs $5 per GB of raw data a month, about $25 for a mid-size project memory. Ingestion takes at most 25 records per call, so a full load of a mid-size project memory is about 16,000 calls.
- **Knowledge organization.** One knowledge base per project memory with metadata on every chunk, as the knowledge organization research recommends.
- **Embedding independence.** The chunk text in the app's own store is the durable layer. The knowledge base's vectors are a cache, and leaving AWS costs one re-embedding run.
- **Graphs.** Left to the graph review. AWS GraphRAG needs a customer-managed knowledge base and S3.
- **Location.** All of it runs in Frankfurt in the consultancy's tenant, which counts as third-party cloud (ADR-0010) and is allowed by the V1 model policy (ADR-0013).
- **Measurement.** The managed Knowledge Base arm in the bake-off.

## Considered options

- Let the Managed Knowledge Base ingest raw files with its own chunking. Rejected because it cannot read GitHub and treats code as plain text, so the arm would measure its chunker instead of its retrieval.
- Use a customer-managed knowledge base with our own vectors. Rejected because it lacks agentic retrieval, the feature the arm is meant to test.
