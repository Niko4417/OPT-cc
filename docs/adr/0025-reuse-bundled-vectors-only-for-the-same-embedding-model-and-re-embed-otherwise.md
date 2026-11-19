---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Reuse bundled vectors only for the same embedding model and re-embed otherwise

Vectors are bound to the embedding model that made them, and chunk text is the durable layer. On import, the installation compares the embedding model of the project memory bundle with its own and takes one of three paths:

1. **Same embedding model.** The importer reuses the bundled vectors.
2. **A different model that the model policy allows.** The importer re-embeds from the bundled chunk text without asking. The project memory is searchable when that finishes. For a mid-size project memory this costs at most about 1.65 dollars on Bedrock in Frankfurt.
3. **No allowed embedding model.** The import stops and names the model policy as the reason.

The importer never searches with vectors from two embedding models at once.

Two additions from the embedding research decide what "same" means and keep word search comparable:

- **Probe.** The exporter stores the vectors of a few fixed probe texts in `space.json`. The importer embeds the same texts with its own model and compares each pair by cosine. If they do not agree within a threshold, the importer treats the model as different. This catches a provider that changes a model behind a stable name, a LiteLLM route that points to another model, and a different quantization of the same model. A hash over the model's metadata sees none of these.
- **Lexical analyzer.** `memory.json` records the tokenizer, the stemmer and its language, the stop-word list and the BM25 parameters. The importer builds its BM25 index with the same settings and warns when it cannot.

## Considered options

- Import in a lexical-only mode when no allowed embedding model exists, as the bundle research proposed. Rejected because the owner ruled out a vectorless mode in the GitHub ingestion review.
- Ask the consultant before re-embedding. Rejected because the model is inside the model policy by definition and the cost is small.
- A hash over model metadata only. Rejected because it cannot see a changed model behind an unchanged name.
- An exact hash of rounded probe vectors, as Keiko does. Rejected because it also breaks on harmless float differences between machines.
- Vector translation between models, SPLADE or ColBERT token vectors in the bundle. Rejected because none removes the dependency on a model, and each adds size or a lossy step.

## Consequences

- The probe does not catch every change. A change that affects only long inputs, or a language the probe texts do not cover, would pass. The probe texts therefore include German, English, prose and code.
- The spec sets the cosine threshold.
- Equal analyzer settings give equal results only on the same search engine. That holds in V1, where every installation runs the same app.
