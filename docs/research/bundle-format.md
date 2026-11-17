# Portable project memory bundle

Date: 2026-11-17. Ticket: [#6](https://github.com/Niko4417/OPT-cc/issues/6), map [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

What do existing tools export when one knowledge base is shared (Keiko Knowledge Pods, the candidate stacks, vector store snapshots)? What must a portable bundle of one project memory contain so another consultant can import it on their machine: source snapshots, chunks, embeddings with their model id, index, model policy, provenance? How do the formats handle a different embedding model on the importing side?

## Summary

No tool in scope exports a knowledge base with its chunks, embeddings and model identity in a portable form. Keiko has no Knowledge Pod export at all. Its own QA ledger says "There is no capsule-export HTTP route in the mounted server inventory". Open WebUI exports a knowledge base as a zip of `.txt` files without vectors. RAGFlow and AnythingLLM only back up whole installations (Docker volumes or the storage folder). Dify's app export leaves linked knowledge bases out. Engine snapshots (Qdrant `.snapshot`, a Lance table directory, a SQLite file) are whole-database copies tied to one engine and, for Qdrant, to one minor version. The map's premise "shared like a Keiko Knowledge Pod" therefore has no existing format to copy. The project memory needs its own bundle format.

Proposal:

1. A bundle is one project memory as a BagIt bag (RFC 8493) inside a zstd-compressed tar file. The payload holds the source snapshot, the chunks as Parquet with full text and source offsets, one Parquet file of vectors per embedding space, the model policy and the ingest provenance. A `memory.json` tag file describes the bundle.
2. Each embedding space records provider, model id, model revision, output dimensions, metric, normalization, instruction text and dtype, plus a SHA-256 fingerprint over those fields. This copies Keiko's `EmbeddingModelIdentity` and its compatibility check.
3. The bundle ships no ANN index and no lexical index. The importer rebuilds both. Index files are engine- and version-specific, and Keiko already treats its ANN index as runtime-local.
4. BagIt SHA-256 manifests cover every file. The exporter signs `tagmanifest-sha256.txt` with `ssh-keygen -Y sign`, so the signature covers the whole bag. The importer checks the signature against the exporter's SSH signing key on GitHub. Optional `age` encryption to the recipients' SSH keys protects client content in transit.
5. On import, vectors are reused only when the embedding-space fingerprint matches the importer's embedding model exactly. Otherwise the importer re-embeds the bundled chunk text with its own model, but only if that model's location is allowed by the bundle's model policy. If no allowed embedding model exists, the memory imports in lexical-only mode and says so. The importer never mixes scores from two embedding spaces.
6. For a mid-size project (30 MB of text, about 20,000 chunks, 1,024-dimension float32 vectors), a bundle is about 150 MB uncompressed and about 90 MB compressed. Vectors are 80 MB of that. A bundle without vectors is about 16 MB.

## 1. What existing tools export

### Keiko Knowledge Pods and Knowledge Pod Sets

Keiko paths below are relative to `/Users/niko/code/Keiko` at commit `b5fe2a84f`.

**Storage.** A Knowledge Pod is the user-facing name for a "capsule". All pods of one namespace live in one SQLite file, `<runtime-state>/local-knowledge/<namespace>/capsules.db` (`docs/local-knowledge/knowledge-pods.md`, table "Product term"; `packages/keiko-local-knowledge/src/store-paths.ts`). The on-disk schema is at version 33 (`packages/keiko-contracts/src/local-knowledge-schema.ts`, `LOCAL_KNOWLEDGE_DB_SCHEMA_VERSION`). The tables that matter for a bundle:

| Table | Content | Relevance |
|---|---|---|
| `capsules` | Pod settings, `model_use_policy_json`, the embedding identity columns, `vector_dimensions`, `vector_metric` | Pod-level manifest |
| `capsule_sources`, `knowledge_sources` | Source scope as `scope_kind` and `scope_json` | `scope_json` holds an absolute `rootPath` (`local-knowledge.ts`, `KnowledgeSourceScope`), so it does not travel to another machine |
| `documents` | Path, size, media type, `content_hash`, parser id and version | Provenance per document |
| `document_blobs` | Original bytes, `storage_kind` plaintext or sealed | Source snapshot |
| `document_texts` | Normalized extracted text | Chunk source |
| `chunks` | Order, token count, `safe_excerpt_hash`, character offsets, chunking strategy version, contextual prefix | Chunks |
| `chunk_lexical_index` plus FTS5 | Plaintext chunk text for BM25 | Rebuildable |
| `vectors` | Float32 `embedding` BLOB plus the full embedding identity on every row | Embeddings |
| `vector_index_state` | State of the native ANN index | The ANN index itself is "runtime-local", not persisted (schema comment above `CREATE_VECTOR_INDEX_STATE`) |
| `repository_file_fingerprints` | Per-file `git-blob-sha1` or file-state fingerprint | Provenance for repository sources |

Chunking defaults are 512 tokens maximum, 64 minimum and 50 tokens overlap (`packages/keiko-local-knowledge/src/chunking/types.ts`), with a budget of 4 characters per token (`chunking/token-estimator.ts`).

**Embedding identity.** `EmbeddingModelIdentity` has `provider`, `modelId`, `vectorDimensions`, `vectorMetric` and the optional `modelRevision`, `normalization`, `instructionVersion`, `embeddingSpaceFingerprint` and `dimensionsParam` (`packages/keiko-contracts/src/local-knowledge.ts`). The compatibility check compares provider, model, revision, model family, dimensions, metric, normalization, instruction version, fingerprint, dimensions parameter and tokenizer, in that order, and reports the first mismatch as a closed reason code (`packages/keiko-contracts/src/local-knowledge-embedding-profiles.ts`, `PROFILE_COMPARISONS`). A pod that lacks the hardening fields is `unknown`, "not silently compatible". It stays usable through lexical retrieval, and Keiko recommends a reindex. Incompatible lanes "fail closed for vector retrieval" (`docs/local-knowledge/knowledge-pods.md`, "Retrieval and score semantics"). Keiko does not reindex private content automatically (`docs/local-knowledge/embedding-space-governance-ledger.md`, "Known limits").

**Model-use policy.** Each pod has a `modelUsePolicy` with mode `standard`, `sealed-local` or `custom`. It resolves to allow or deny for seven operations: `externalEmbeddings`, `localEmbeddings`, `externalReranking`, `localReranking`, `answerSynthesis`, `rawContentRelease` and `evidencePersistence` (`packages/keiko-contracts/src/local-knowledge-model-use-policy.ts`). When several pods combine, "deny wins per operation" (`knowledge-pods.md`, "Model-use policy"). This maps onto the glossary's model policy, which is one model location limit instead of seven switches.

**Encryption.** When a key provider is present, the store seals content rows. The key comes from `KEIKO_LOCAL_KNOWLEDGE_KEY`, then a keychain service, then a `0600` keyfile next to `capsules.db` (`docs/adr/ADR-0047-local-knowledge-content-encryption.md`, D2). FTS5 text stays plaintext because "SQLite FTS5 cannot index sealed randomized envelopes" (schema comment above `CREATE_CHUNK_LEXICAL_INDEX`). A copied `capsules.db` is therefore unreadable without the key, and its FTS rows are plaintext anyway.

**"No global pool".** Keiko calls this a Foundry-IQ invariant. Every document, chunk and vector row carries `capsule_id`, and chunks and vectors also carry `source_id` and `document_id`, with composite foreign keys. The schema comment says "a chunk or vector cannot exist outside of its capsule + source + document tuple" (`local-knowledge-schema.ts`, "Lineage invariant"). A Knowledge Pod Set "deliberately holds only capsule IDs, no vectors, no chunks, no documents" (`local-knowledge.ts`, `CapsuleSet`). Search runs per pod and merges with rank fusion, never by raw vector score (ADR-0036, `docs/adr/ADR-0036-hybrid-grounding-reciprocal-rank-fusion.md`). For sharing this means: a bundle is exactly one project memory with its own lineage ids, an import never merges it into another memory's chunks or vectors, and sets of memories are references that do not travel in a bundle.

**Export and import.** There is none. The Jira Pods QA record states "There is no capsule-export HTTP route in the mounted server inventory" and calls its test "an absence/refusal proof, not a claim that Data Pods support export" (`docs/qa/jira-chat-pods-4081.md`, #4085 AC7). The mounted local-knowledge routes are `capsules`, `capsule-sets`, `citation-preview` and `manual-pods`, none of them export or import (`packages/keiko-server/src/local-knowledge-handlers.ts` and siblings). `knowledge-pods.md` lists "shared" pods as out of scope and requires an ADR first ("Future remote, federated, or ephemeral pods"). Copying `capsules.db` is not a pod export either: it holds every pod of the namespace, absolute source paths and, if encrypted, content that needs the sender's key.

What Keiko does give the bundle design: the embedding identity fields, the compatibility check with closed reasons, lexical fallback, per-pod lineage, the policy operations, and git blob SHA-1 fingerprints for repository files.

### Candidate stacks

Stacks from `docs/research/rag-stack-landscape.md` on branch `research/rag-stack-landscape`.

| Stack | Export of one knowledge base | Vectors included | On embedding model change |
|---|---|---|---|
| Open WebUI | Admins export a knowledge base as a zip; "Files are converted to `.txt`" ([knowledge docs](https://docs.openwebui.com/features/workspace/knowledge.md)). Chat import exists, knowledge import does not ([import and export](https://docs.openwebui.com/features/chat-conversations/data-controls/import-export.md)) | No | "requires a re-index" that deletes the collection and parses every file again ([RAG docs](https://docs.openwebui.com/features/chat-conversations/rag.md)) |
| RAGFlow | None. `docker/migration.sh` backs up and restores the volumes `mysql_data`, `minio_data`, `redis_data` and `esdata01` of a whole deployment ([script](https://github.com/infiniflow/ragflow/blob/main/docker/migration.sh)) | Only inside a full Elasticsearch volume | Not checked |
| AnythingLLM | None found. The storage folder holds `anythingllm.db`, `documents`, `lancedb`, `vector-cache` and `models` ([storage docs](https://docs.anythingllm.com/installation-desktop/storage)) | Only by copying the whole folder | Switching vector database "will not automatically port over your already embedded information" ([vector DB docs](https://docs.anythingllm.com/setup/vector-database-configuration/overview)) |
| Dify | App export (`.ifpkg`, a ZIP with `app.yaml`) excludes "Linked knowledge bases" ([app management](https://docs.dify.ai/en/guides/management/app-management)). Knowledge pipeline DSL exports configuration only. A request for knowledge base export is open as a discussion ([#4394](https://github.com/langgenius/dify/discussions/4394)) | No | Not checked |
| Onyx | No export or backup guide found. Docs cover Vespa to OpenSearch index migration inside one install ([OpenSearch migration](https://docs.onyx.app/admins/advanced_configs/opensearch_document_index_migration)) | No | Not checked |
| LightRAG | No export. Storage is files under `WORKING_DIR` or an external database ([README](https://github.com/HKUDS/LightRAG/blob/main/README.md)) | Only by copying the working directory | "Changing it requires re-embedding text chunks, entities, and relationships." The `lightrag-rebuild-vdb` tool rebuilds vectors from the stored chunks and graph, and the README says not to delete the working directory instead (README, "Embedding Models") |

Two patterns matter. First, every stack treats a different embedding model as "re-embed everything". Second, LightRAG re-embeds from stored chunk text, while Open WebUI parses the source files again. Shipping chunk text in the bundle allows the faster LightRAG path and keeps chunk ids, and therefore citations, stable across machines.

### Vector store snapshot formats

| Format | What it is | Portability limits |
|---|---|---|
| Qdrant collection snapshot | A `tar` file with extension `.snapshot`, "including the collection configuration, all points and payloads". Aliases are excluded, and each node is snapshotted separately ([snapshots](https://qdrant.tech/documentation/snapshots/)) | Restores "can only be restored to other Qdrant clusters running the same minor version or the next minor version" (same page). Recovery accepts an optional "SHA256 checksum to verify snapshot integrity before recovery" ([OpenAPI `SnapshotRecover`](https://github.com/qdrant/qdrant/blob/master/docs/redoc/master/openapi.json)) |
| Lance table (LanceDB) | A directory with `data/` (`*.lance`), `_versions/` (one manifest per version), `_deletions/`, `_indices/` (one folder per index UUID), `_transactions/` and `_refs/`. Copying the root works without manifest changes because file references are relative, except for tables with several base paths ([layout spec](https://lance.org/format/table/layout/)) | Readable only by Lance-format readers. Indexes live in `_indices/` and follow the Lance version |
| SQLite file (Keiko, AnythingLLM's `anythingllm.db`, sqlite-vec) | One file. sqlite-vec stores float, int8 and binary vectors in `vec0` virtual tables and is "pre-v1, so expect breaking changes" ([README](https://github.com/asg017/sqlite-vec)) | Copying a live file is unsafe; the [backup API](https://sqlite.org/backup.html) or `VACUUM INTO` makes a consistent copy. The schema is app-specific |

None of these records the embedding model. A Qdrant snapshot knows vector size and distance, not which model produced the vectors. An engine snapshot is a good backup of one installation and a poor exchange format between two.

### Open packaging formats

- **BagIt** ([RFC 8493](https://www.rfc-editor.org/rfc/rfc8493.html)). A bag has `bagit.txt`, a `data/` payload directory and at least one `manifest-<alg>.txt` with a checksum per payload file. Optional tag files are `bag-info.txt` (metadata, including `Payload-Oxum` for size), `tagmanifest-<alg>.txt` and `fetch.txt`. Tools must support SHA-256 and SHA-512. A bag is "valid" when it is complete and every checksum verifies. BagIt is "not designed to be secure against active attacks" and leaves signatures to the users.
- **Frictionless Data Package** ([data resource spec](https://datapackage.org/standard/data-resource/)). A resource descriptor has `path`, `bytes` and `hash`, with MD5 as default and other algorithms by prefix, for example `sha256:...`. Useful as a model for describing tables, but BagIt already covers hashes.
- **Apache Arrow and Parquet.** Arrow's fixed-size list type is "a nested type in which each array slot contains a fixed-size sequence of values all having the same type" ([columnar format](https://arrow.apache.org/docs/format/Columnar.html#fixed-size-list-layout)). A vector column as `FixedSizeList<float32>[1024]` in Parquet is readable by Lance, DuckDB, pandas and pyarrow without a vector database.

No open format for "a RAG knowledge base with its embeddings" turned up. Combining BagIt for integrity with Parquet for tables uses two well-known standards and needs no new container.

## 2. What goes in a bundle

| Part | Include | Why |
|---|---|---|
| Source snapshot | Yes. Repo files at one commit, and issues and pull requests as JSON from the [GitHub REST API](https://docs.github.com/en/rest/issues/issues) | Citations open the cited text on the importer's machine. Re-chunking with a different chunker needs the sources. GitHub access on the importing side is not guaranteed |
| Chunks | Yes, with full text, source id, repo-relative path, document hash, character and line offsets, chunker id and version | Re-embedding without re-parsing (the LightRAG path). Stable chunk ids keep citations identical |
| Embeddings | Yes, one file per embedding space, with the full identity next to it | Skips re-embedding when the importer has the same model |
| ANN index (HNSW, IVF) | No. Record only the index parameters | Engine- and version-specific (Qdrant minor version rule, Lance `_indices/`). Keiko keeps its ANN index runtime-local. Rebuilding from 20,000 vectors is a local step |
| Lexical index | No | Rebuilt from chunk text. Keiko also derives its FTS5 table from chunk rows through triggers |
| Model policy | Yes | The importer must apply the same limit or a stricter one |
| Provenance | Yes. Repo, commit SHA, git blob SHA-1 per file, issue and PR numbers with `updated_at`, fetch time, parser, chunker and embedding versions, exporter app and version, exporter identity | Lets the importer judge freshness and lets a later re-sync work incrementally (re-sync itself is open on the map) |
| Content hashes | Yes. SHA-256 per payload file (BagIt) and per chunk text | Integrity and deduplication |
| Credentials, absolute paths, local settings | Never | Keiko's `scope_json` shows why: an absolute `rootPath` is meaningless on another machine and leaks the sender's home directory |
| Chat history and answers | No | Not part of the project memory as the glossary defines it |

## 3. Integrity and signing

- **Integrity.** BagIt `manifest-sha256.txt` lists every payload file. `tagmanifest-sha256.txt` lists the tag files, including `memory.json` and `manifest-sha256.txt`. An importer checks the bag is valid before reading any content. SHA-256 also matches what Qdrant uses for its snapshot checksum.
- **Signing.** Consultants already have SSH keys for GitHub. `ssh-keygen -Y sign -n <namespace> -f <key> tagmanifest-sha256.txt` writes `tagmanifest-sha256.txt.sig`. `ssh-keygen -Y verify` checks it against an `allowed_signers` file with an identity and a namespace restriction, and returns exit status 0 on success ([ssh-keygen(1)](https://man.openbsd.org/ssh-keygen)). Because the tag manifest covers `manifest-sha256.txt`, which covers every payload file, one signature covers the whole bag. The importer can build `allowed_signers` from `GET /users/{username}/ssh_signing_keys`, which "is accessible by anyone" ([GitHub API](https://docs.github.com/en/rest/users/ssh-signing-keys)). Sigstore keyless signing needs an OpenID Connect identity at signing time ([cosign blob signing](https://docs.sigstore.dev/cosign/signing/signing_with_blobs/)), which fits less well with local-first use.
- **Confidentiality.** A bundle holds client content. `age` encrypts to several recipients and accepts SSH public keys ("ssh-ed25519 AAAA...") as recipients ([age README](https://github.com/FiloSottile/age)). Encrypting the `.pmb` file to the named consultants' SSH keys needs no new key infrastructure. Unlike a Keiko store, the bundle does not depend on the sender's local key.

## 4. Size estimate for a mid-size repo

Assumptions: a mid-size modernization project has 2,500 repo files with 20 MB of text, plus 3,000 issues and pull requests with 10 MB of text, so 30 MB in total. Chunking follows Keiko's defaults (512 tokens, 50 overlap, 4 characters per token, so 2,048 characters per chunk and a 1,848-character step). The embedding model has 1,024 dimensions, the same as Qwen3-Embedding-0.6B ([model card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B)) and BGE-M3 (`hidden_size` 1024 in its [config](https://huggingface.co/BAAI/bge-m3/blob/main/config.json)).

| Part | Calculation | Uncompressed | Compressed |
|---|---|---|---|
| Source snapshot | 30 MB text | 30 MB | 7 MB |
| Chunks | 30 MB / 1,848 characters is about 16,000 chunks; at least one chunk per file or issue raises this to about 20,000. Text with 10% overlap is 33 MB | 33 MB plus 3 MB metadata | 8 MB |
| Vectors | 20,000 x 1,024 x 4 bytes | 82 MB | 76 MB |
| Policy, provenance, manifests | | under 2 MB | under 1 MB |
| Total | | about 150 MB | about 90 MB |

Measured compression ratios: the 9,167 text files under 200 KB in the Keiko repo (125 MB) compress to 28.7 MB with `zstd -3`, a ratio of 4.3. 2,000 synthetic unit vectors of 1,024 float32 values compress by 7% with `zstd -19`. Real embeddings may compress slightly better; plan for the raw size.

Two cheaper variants exist. Storing vectors as float16 halves them to 41 MB. Leaving vectors out gives a 16 MB bundle that the importer must re-embed. For a repo the size of Keiko (125 MB of text), the same calculation gives about 70,000 chunks and 290 MB of float32 vectors. Re-embedding time depends on the embedding throughput that ticket #4 measures.

## 5. Import with a different embedding model

The importer compares the bundle's embedding-space fingerprint with the identity of its own configured embedding model, field by field as Keiko's `PROFILE_COMPARISONS` does. Then:

1. **Same fingerprint.** Load the bundled vectors and build the ANN index locally.
2. **Different fingerprint, and the importer's embedding model is allowed by the bundle's model policy.** Re-embed the bundled chunk text with the importer's model and record a new embedding space. Keep the chunk ids. Do not use the bundled vectors for search, and never compare scores across the two spaces (Keiko ADR-0036). Re-parsing the sources is needed only if the importer also wants a different chunker.
3. **Different fingerprint, and no allowed embedding model.** Import in lexical-only mode and show that dense search is unavailable, as Keiko does for `unknown` and `incompatible` pods. Do not re-embed through a model whose location the policy forbids.
4. **Refuse the import** when the bag is not valid, the signature fails or the signer is not trusted, `format_version` has an unknown major version, or the model policy has a value the importer does not know.

The importer may make the model policy stricter, never looser. Re-embedding with an on-device model is always allowed under the default on-device-only policy. Re-embedding on import is a local, explicit step that the consultant starts, in line with Keiko's rule against automatic reindexing of private content.

## 6. Proposed bundle layout

File name `<memory-slug>-<yyyy-mm-dd>.pmb`: a tar archive compressed with zstd, optionally wrapped with `age` as `<name>.pmb.age`. Inside, one BagIt bag:

```
acme-billing-2026-11-17/
  bagit.txt                      BagIt-Version: 1.0, UTF-8
  bag-info.txt                   Bagging-Date, Payload-Oxum, External-Identifier = memory id
  memory.json                    bundle manifest (below)
  manifest-sha256.txt            SHA-256 of every file under data/
  tagmanifest-sha256.txt         SHA-256 of bagit.txt, bag-info.txt, memory.json, manifest-sha256.txt
  tagmanifest-sha256.txt.sig     ssh-keygen -Y sign, namespace "project-memory-bundle"
  data/
    policy/model-policy.json     {"max_model_location": "on-device"}
    sources/
      repos/acme/billing/        repo files at the recorded commit, repo-relative paths
      issues/acme/billing/1234.json
      pulls/acme/billing/1301.json
    chunks/chunks.parquet        one row per chunk
    embeddings/
      e1/space.json              embedding identity and fingerprint
      e1/vectors.parquet         chunk_id, vector FixedSizeList<float32>[1024]
    provenance/sources.json      repo, commit, blob SHA-1 per file, issue and PR numbers with updated_at
    provenance/ingest-runs.jsonl one line per ingest run: tool versions, counts, start and end time
```

`chunks.parquet` columns: `chunk_id`, `source_id`, `document_path`, `document_sha256`, `git_blob_sha1` (repo files only), `char_start`, `char_end`, `line_start`, `line_end`, `order_index`, `token_count`, `text`, `text_sha256`, `chunker_id`, `chunker_version`.

`memory.json`:

```json
{
  "format": "project-memory-bundle",
  "format_version": "1.0",
  "memory_id": "7c1e9a52-3b4d-4f7e-9a51-0d2f6c8b1e44",
  "name": "ACME billing modernization",
  "created_at": "2026-11-17T15:00:00Z",
  "exporter": { "app": "<app name>", "version": "<app version>" },
  "exported_by": { "github_login": "<login>", "ssh_key_fingerprint": "SHA256:<fingerprint>" },
  "model_policy": "data/policy/model-policy.json",
  "sources": [
    { "id": "s1", "kind": "github-repo", "repo": "acme/billing", "ref": "main", "commit": "<40-char sha>" },
    { "id": "s2", "kind": "github-issues", "repo": "acme/billing", "fetched_at": "2026-11-17T14:40:00Z", "count": 2100 },
    { "id": "s3", "kind": "github-pulls", "repo": "acme/billing", "fetched_at": "2026-11-17T14:40:00Z", "count": 900 }
  ],
  "chunking": { "chunker_id": "<id>", "chunker_version": "<version>", "max_tokens": 512, "overlap_tokens": 50 },
  "embedding_spaces": ["e1"],
  "index_hints": { "metric": "cosine", "ann": "hnsw", "m": 16, "ef_construction": 200 },
  "counts": { "documents": 5500, "chunks": 20000, "vectors": { "e1": 20000 } }
}
```

`embeddings/e1/space.json`:

```json
{
  "id": "e1",
  "provider": "local-mlx",
  "model_id": "Qwen/Qwen3-Embedding-0.6B",
  "model_revision": "<Hugging Face commit sha>",
  "dimensions": 1024,
  "dimensions_param": null,
  "metric": "cosine",
  "normalization": "l2",
  "dtype": "float32-le",
  "document_instruction": "",
  "query_instruction": "Instruct: <task description>\nQuery:",
  "tokenizer": "<tokenizer id and revision>",
  "model_location": "on-device",
  "fingerprint": "sha256:<hash of the canonical JSON of the fields above>"
}
```

## Open points

- The glossary has no term for the exported file yet. "Project memory bundle" is a candidate for `GLOSSARY.md`.
- Deleting content from exported copies and re-syncing after repo changes stay open on the map. The provenance block (commit SHA, blob SHA-1, `updated_at`) is what an incremental re-sync would need.
- Whether the chosen stack (#10) can load chunks and vectors from Parquet, or needs glue code to write them into its own store, depends on that choice.
- Whether workshop results and decision records need more than repo-relative paths in provenance depends on #9.

## Sources

Keiko, local checkout `/Users/niko/code/Keiko` at `b5fe2a84f`:

- `docs/local-knowledge/knowledge-pods.md`
- `docs/local-knowledge/embedding-space-governance-ledger.md`
- `docs/qa/jira-chat-pods-4081.md`
- `docs/adr/ADR-0036-hybrid-grounding-reciprocal-rank-fusion.md`
- `docs/adr/ADR-0047-local-knowledge-content-encryption.md`
- `packages/keiko-contracts/src/local-knowledge.ts`
- `packages/keiko-contracts/src/local-knowledge-schema.ts`
- `packages/keiko-contracts/src/local-knowledge-embedding-profiles.ts`
- `packages/keiko-contracts/src/local-knowledge-model-use-policy.ts`
- `packages/keiko-local-knowledge/src/store-paths.ts`
- `packages/keiko-local-knowledge/src/chunking/types.ts`, `chunking/token-estimator.ts`
- `packages/keiko-local-knowledge/src/composition.ts`, `capsule-set-lifecycle.ts`

Prior research: `docs/research/rag-stack-landscape.md` on branch `research/rag-stack-landscape`.

External:

- Open WebUI: [knowledge](https://docs.openwebui.com/features/workspace/knowledge.md), [RAG](https://docs.openwebui.com/features/chat-conversations/rag.md), [import and export](https://docs.openwebui.com/features/chat-conversations/data-controls/import-export.md)
- RAGFlow: [docker/migration.sh](https://github.com/infiniflow/ragflow/blob/main/docker/migration.sh)
- AnythingLLM: [storage](https://docs.anythingllm.com/installation-desktop/storage), [vector databases](https://docs.anythingllm.com/setup/vector-database-configuration/overview)
- Dify: [app management](https://docs.dify.ai/en/guides/management/app-management), [discussion #4394](https://github.com/langgenius/dify/discussions/4394)
- Onyx: [OpenSearch index migration](https://docs.onyx.app/admins/advanced_configs/opensearch_document_index_migration)
- LightRAG: [README](https://github.com/HKUDS/LightRAG/blob/main/README.md)
- Qdrant: [snapshots](https://qdrant.tech/documentation/snapshots/), [OpenAPI spec](https://github.com/qdrant/qdrant/blob/master/docs/redoc/master/openapi.json)
- Lance: [table layout](https://lance.org/format/table/layout/)
- sqlite-vec: [README](https://github.com/asg017/sqlite-vec); SQLite: [backup API](https://sqlite.org/backup.html)
- BagIt: [RFC 8493](https://www.rfc-editor.org/rfc/rfc8493.html); Frictionless: [data resource](https://datapackage.org/standard/data-resource/); Arrow: [columnar format](https://arrow.apache.org/docs/format/Columnar.html#fixed-size-list-layout)
- Signing and encryption: [ssh-keygen(1)](https://man.openbsd.org/ssh-keygen), [GitHub SSH signing keys API](https://docs.github.com/en/rest/users/ssh-signing-keys), [cosign blob signing](https://docs.sigstore.dev/cosign/signing/signing_with_blobs/), [age](https://github.com/FiloSottile/age)
- Models: [Qwen3-Embedding-0.6B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B), [BGE-M3 config](https://huggingface.co/BAAI/bge-m3/blob/main/config.json)
- GitHub: [REST issues API](https://docs.github.com/en/rest/issues/issues)
