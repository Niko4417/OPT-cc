---
status: accepted
date: 2026-11-20
decided-by: Niko4417
ticket: https://github.com/Niko4417/OPT-cc/issues/21
---

# Pack a project memory bundle as a BagIt bag with Parquet tables and no search index

No tool in scope exports one knowledge base with its chunks, vectors and embedding model identity, so the project memory bundle is our own format. A bundle is one BagIt bag (RFC 8493) in a zstd-compressed tar file with the extension `.pmb`. It holds exactly one project memory, and an import never mixes it into another project memory's chunks or vectors.

What a bundle holds:

- **Manifest.** `memory.json` names the format and its `format_version`, the project memory's id and name, the exporter, the sources with their pinned commits, the chunk store's schema version, the lexical analyzer and the embedding model (ADR-0025). The importer refuses an unknown major `format_version`.
- **Chunks.** `chunks.parquet` is the app's chunk store table, exported as it is. The bundle format requires only four columns, the ones import itself depends on: chunk id, chunk text, text hash and source file path. Every other column passes through untouched and belongs to the ingestion decisions (ADR-0018, ADR-0019 and the folder and document ingestion review).
- **Vectors.** Always included, for the installation's current embedding model only, as float32 in one Parquet file next to a `space.json` with that model's identity. There is no option to leave them out. The vectors inside the Bedrock Managed Knowledge Base cannot be exported and never travel. The importer rebuilds that knowledge base from chunk text (ADR-0017).
- **No search index.** The importer builds its own vector index and BM25 index.
- **Plain files, history, model policy and provenance.** See ADR-0022, ADR-0027 and ADR-0026.
- **Integrity.** BagIt SHA-256 manifests cover every file. Signing and encryption are in ADR-0023.

A bundle never holds credentials (the GitHub token, API keys), LiteLLM routes, installation settings, absolute paths, chat history or answers. After an import, the importer connects their own GitHub token before they can sync the repository again. The bundle names its exporter by the fingerprint of the installation key and a display name the consultant chose.

Illustration of the layout. The spec fixes the folder names.

```text
<memory-slug>-<yyyy-mm-dd>/
  bagit.txt
  bag-info.txt
  memory.json
  manifest-sha256.txt
  tagmanifest-sha256.txt
  tagmanifest-sha256.txt.sig
  data/
    policy/model-policy.json
    sources/           working tree, github/ Markdown files, documents, transcripts
    records.bundle     git bundle of the human-made text with its history
    chunks/chunks.parquet
    embeddings/<id>/space.json
    embeddings/<id>/vectors.parquet
    provenance/
```

Answers to the standing architecture questions:

- **Retrieval per source type.** Does not apply to the format. A bundle ships no index. It carries the working tree and the chunks, so the importer can rebuild every retrieval method the code retrieval review chooses.
- **Scale.** A mid-size project memory (30 MB of text, about 20,000 chunks, 1,024-dimension vectors) gives about 150 MB uncompressed and 90 MB compressed, of which vectors are about 76 MB. A repository with 125 MB of text gives about 70,000 chunks and 290 MB of vectors. Original document files and recordings come on top (ADR-0022).
- **Knowledge organization.** One bundle per project memory. Chunk metadata travels, so one index or several can be built from it.
- **Embedding independence.** Chunk text is the durable layer and vectors are a cache (ADR-0025).
- **Graphs.** Not shipped. The importer rebuilds them without a language model (ADR-0022).
- **Location.** The format has nothing specific to AWS. Only the key storage (ADR-0023) and the transport (ADR-0024) use AWS services, and both are per installation.
- **Measurement.** The trial run exports a project memory and imports it into a second installation.

## Considered options

- An engine snapshot (Qdrant snapshot, Lance table, SQLite file). Rejected because each is bound to one engine and version, none records the embedding model, and the base stack is still open.
- JSON Lines for chunks and vectors. Rejected because it has no types and a vector as text takes about three times its float32 size.
- A SQLite file. The closest runner-up. Rejected because its schema is app-specific and vectors are untyped BLOBs. Its strength is updating in place, which a bundle never does.
- `.npy` or safetensors for vectors. Rejected because vectors and chunk ids would sit in separate files, matched only by row position.
- A zip with our own manifest of hashes in place of BagIt. Rejected because BagIt already defines what a complete and valid package is, has maintained validators, and gives one tag manifest to sign.
- An OCI artifact in a registry such as Amazon ECR. Rejected for V1 because two installations would need a shared registry (ADR-0015). The same bag can be pushed as an OCI artifact later.
- Zip as the outer file. Rejected because tar keeps the symlinks and executable bits of a repository working tree and can be streamed to and from S3. Size does not decide between the two.
- A column list owned by the bundle format. Rejected because three other topics add columns, and a column forgotten in a mapping would drop citation data without an error.
- Bundles without vectors, or with float16 vectors. Rejected to keep one path. Vectors add about 76 MB.

## Consequences

- The bundle is tied to the app's chunk store schema. An importer has to read older schema versions and migrate them. Other tools can still open the file, because Parquet stores column names and types.
