---
status: accepted
date: 2026-11-19
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/20
---

# Ingest GitHub repositories with our own glue job into our own chunk store

No candidate app ingests code, issues, pull requests and review comments together with line-range citations, and no managed knowledge base on AWS or Microsoft has a GitHub connector. So one glue ingestion job that this project owns reads every connected GitHub repository. It writes chunks with their metadata to the app's own chunk store, which is the durable layer. A small loader per bake-off arm pushes those chunks into that arm's index: Keiko, Open WebUI, LightRAG, HippoRAG 2, and the Bedrock Managed Knowledge Base through its custom connector (ADR-0017). Every arm therefore reads the same ingestion output.

The job keeps one git clone per connected repository on AWS in Frankfurt and records the commit SHA of every ingestion run. In V1 it reads only the default branch head, with no other branches, tags or forks. The clone also gives the agent arms their grep and line-range file reads. Code therefore reaches the project memory in two forms: the pinned clone and its chunks. How each form is searched is left to the code retrieval review.

On every sync, the job also writes each issue and pull request as one Markdown file under a `github/` folder next to the clone, for example `github/issues/1234.md`. The file holds the body, conversation comments, review bodies and review comments in order, each under a heading with its author, date and own `html_url`. Front matter holds the number, title, state, labels and the item's `html_url`. The chunk records are cut from these files. A project memory is then complete as plain files, so any harness can read it with grep and file reads, with no embedding model and no GitHub access.

The job reads GitHub with a fine-grained personal access token per project memory, read-only for Contents, Issues, Pull requests and Metadata, stored in AWS Secrets Manager in Frankfurt. It uses no webhooks.

Answers to the standing architecture questions:

- **Retrieval per source type.** Code arrives as the pinned clone and as tree-sitter chunks. Prose arrives as heading chunks, and GitHub items as one record each (ADR-0019). How each form is searched is left to the code retrieval review.
- **Scale.** A full load of the test dataset (`thunderbird/thunderbird-android`: 8,601 files, 6,304 issues, 5,507 pull requests, about 28,400 conversation comments and 6,600 review comments) takes about 6,000 GitHub calls. That is about 470 list pages plus about 5,500 calls for review bodies, or about 1.2 hours at 5,000 calls per hour. A nightly incremental run takes a few dozen calls. Chunking needs no model, and Magika adds about 5 ms per file it checks.
- **Knowledge organization.** Every chunk carries `source_type` (code, doc, issue, pull request, conversation comment, review comment, review body), repository, number, path, commit SHA, line range and `vendored` or `generated` tags. One index or several is left to the knowledge organization review. Either layout can be built from the same chunks without ingesting again.
- **Embedding independence.** Ingestion writes text chunks and calls no embedding model. Each arm's indexing step embeds them later with the model the model review chooses, or with the Bedrock knowledge base's own model. Vectors are a cache keyed by chunk text hash and embedding model (ADR-0020).
- **Graphs.** The static code graph could reuse the same tree-sitter parse. Left to the graph reviews.
- **Location.** The clone, the `github/` files, the chunk store and the token sit in the consultancy's AWS tenant in Frankfurt. Magika and go-enry run inside the job. Ingestion sends no content to any language, embedding or reranking model, so the model policy first applies when chunks are embedded or used to answer.
- **Measurement.** Every bake-off arm reads the same ingestion output, so ingestion is not a variable between arms. Gold questions that cite code lines, issues, pull requests and review comments test the citations.

## Considered options

- Each stack's own GitHub connector. Rejected because Onyx reads no code or comments, RAGFlow reads only issue and pull request bodies, Dify caps threads at 30 comments and skips review comments, and the others read no issues at all. None cites a line range.
- Let the glue job write straight into each app's store. Rejected because every arm should read from one place, and the chunk text must survive a change of app or embedding model.
- A GitHub App instead of a personal access token. Rejected for V1, which is single-user (ADR-0015). It returns with a shared team server.
- Keep issues and pull requests only as chunks. Rejected because a harness at a client without GitHub access could not read them.

## Consequences

- The project memory bundle review should carry the plain-file layout next to the chunks.
- Sourcebot is not used. It became closed source with v6.
