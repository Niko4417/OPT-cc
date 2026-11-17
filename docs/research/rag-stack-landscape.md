# On-premise RAG stacks for a project memory

Date: 2026-11-17

## Question

Which open-source, self-hostable RAG stack is the best base for a reference installation of a "project memory" that combines source code, documentation, workshop results and decision records of a software modernization project and answers natural-language questions with sources? The installation must run fully on-premise for German and EU clients under GDPR, and this note also covers code retrieval, decision records, local models, evaluation, test datasets and privacy requirements.

## Summary

Recommendation for the reference installation:

1. **Onyx Community Edition as the main application, plus a separate code index.** Onyx has the broadest connector list that matches the sources in this project: GitHub, GitLab, Bitbucket, Confluence, Jira, SharePoint and file upload ([connector overview](https://docs.onyx.app/admins/connectors/overview)). It filters every search by the user's ACL ([access controls](https://docs.onyx.app/security/architecture/access_controls)) and has an air-gapped mode ([self-hosted FAQ](https://docs.onyx.app/security/self_hosted/faq)). It ships Docker Compose and Helm ([deployment folder](https://github.com/onyx-dot-app/onyx/tree/main/deployment)). Release v5.1.3 is from 2026-11-16 ([release](https://github.com/onyx-dot-app/onyx/releases/tag/v5.1.3)). The limitation is licensing. Permission sync and user groups are Business-tier features ([editions](https://docs.onyx.app/deployment/miscellaneous/enterprise_edition)). In the Community Edition, a document is either public to all users or visible only to the connector owner. For a reference setup, run one Onyx instance per client project. The DSK RAG guidance names client separation ("Mandantentrennung") as a measure ([DSK RAG guidance, p. 11-12](https://datenschutzkonferenz-online.de/media/oh/DSK_OH_RAG.pdf)).
2. **RAGFlow 1.0 as the fully Apache-2.0 alternative.** It has no paid tier, a strong document parser and connectors for Confluence, Jira, SharePoint, GitLab (including code files) and GitHub (issues and pull requests) ([connector sources](https://github.com/infiniflow/ragflow/tree/main/internal/syncer/connector)). It also has "Knowledge Compilation" that generates timelines and wikis ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). Access control is limited to "only me" or "team" per dataset, with no ACL sync from source systems ([sharing docs](https://github.com/infiniflow/ragflow/blob/main/docs/guides/team/resource_sharing/resource_sharing_and_permissions.md)). Version 1.0.0 (2026-11-06) is a Go rewrite and was 11 days old on the date of this note ([release](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0)).

For code, neither stack does AST-aware chunking. Add Sourcebot (MIT, self-hosted code search with an MCP server and symbol navigation) as a second index, and connect it as a tool ([README](https://github.com/sourcebot-dev/sourcebot)). Use Ollama or llama.cpp on Apple Silicon and vLLM on a single NVIDIA GPU. Start with Qwen3-Embedding or BGE-M3 for embeddings, bge-reranker-v2-m3 or Qwen3-Reranker for reranking, and Qwen3.6-35B-A3B or gpt-oss-20b as the LLM. All of these are Apache-2.0 or MIT.

The "C4" in the task most likely means codecentric's "c4 GenAI Suite". That GitHub repository was archived on 2026-10-23, so it should not be the base (details in section 1).

PrivateGPT 1.0 is now an API layer without user management, OAuth or source connectors ([README](https://github.com/zylon-ai/private-gpt)). It is a possible backend, not an end-user project memory.

Open decisions:

- Use one instance per client project (Onyx CE is enough), or one shared instance with document-level ACLs (needs an Onyx Business license, or Open WebUI groups).
- Pick the inference hardware: one 24-48 GB NVIDIA GPU with vLLM, or Apple Silicon with Ollama or llama.cpp. Choose the LLM by a German-language evaluation on the test dataset.
- Decide how deep code retrieval goes: text chunks in the document RAG only, a code search index through MCP, or a code knowledge graph.
- Model decision records with metadata conventions (MADR front matter plus filters), or add a temporal graph (Graphiti).
- Choose the test data: a public project (Backstage, GOV.UK infrastructure, Apache Kafka), or client data, which needs a legal basis, a DPIA check and possibly works council involvement.

## 1. Candidate stacks

### What "C4" most likely refers to

There is an open-source RAG product named "c4": the [c4 GenAI Suite](https://github.com/codecentric/c4-genai-suite) by codecentric AG, a German IT consultancy. The evidence is as follows.

- It is a self-hostable chat application with assistants, extensions and MCP support. Its own RAG server, REI-S, uses pgvector or Azure AI Search, OpenAI-compatible or Ollama embeddings, and parses PDF, DOCX, PPTX and XLSX ([README](https://github.com/codecentric/c4-genai-suite/blob/main/README.md)).
- It ships a Docker Compose file with a bundled Ollama and a Helm chart. The Helm chart has OpenID Connect settings ([Helm README](https://github.com/codecentric/c4-genai-suite/blob/main/helm-chart/README.md)).
- It has a proof-of-concept Confluence importer and an eval service ([services folder](https://github.com/codecentric/c4-genai-suite/tree/main/services)).
- It is Apache-2.0 licensed and is named next to PrivateGPT in a German consulting context, which fits the task wording.

Status: the GitHub API reports the repository as archived. The last release, v10.0.4 on 2026-10-23, was a Node.js CVE fix ([release](https://github.com/codecentric/c4-genai-suite/releases/tag/v10.0.4)). I found no announcement of a successor. The former English product page URL now shows a generic codecentric GenAI page with no mention of c4 (unverified whether this is a redirect). The [C4 model](https://c4model.com/) by Simon Brown is an architecture diagram notation, not a RAG stack. It is still useful as a structure for architecture documentation in the corpus. Conclusion: "C4" almost certainly means the c4 GenAI Suite, which is now an archived project and not a maintained base.

### Comparison table

Release dates and licenses come from the GitHub API on 2026-11-17. "CE" means community edition.

| Stack | License (gated parts) | Latest release | Local LLM and embeddings | Relevant connectors | Access control and SSO | Citations | Deployment | Status |
|---|---|---|---|---|---|---|---|---|
| c4 GenAI Suite | Apache-2.0 | v10.0.4, 2026-10-23 | Ollama, OpenAI-compatible | File upload, Confluence importer (PoC) | User groups, OIDC | Not checked | Compose, Helm | Archived |
| PrivateGPT 1.0 | Apache-2.0 | v1.0.1, 2026-06-18 | Any OpenAI-compatible server | File ingestion, MCP, databases | Token auth only, no OAuth or organizations | Yes | pip/brew, Docker | Active commits, API layer only |
| Onyx | MIT, except `ee` folders (Onyx Enterprise License) | v5.1.3, 2026-11-16 | Ollama, self-hosted embedding model server | GitHub, GitLab, Bitbucket, Confluence, Jira, SharePoint, Teams, file upload | CE: one OIDC or SAML provider, docs public or owner-only. Business: permission sync, groups | Yes | Compose, Helm, air-gap flag | Very active |
| RAGFlow | Apache-2.0 | v1.0.0, 2026-11-06 | Many providers incl. local, see notes | Confluence, Jira, SharePoint, OneDrive, GitLab (code files), GitHub (issues, PRs), S3, IMAP | Dataset sharing "only me" or "team"; SSO not confirmed | Yes | Compose | Very active, fresh rewrite |
| AnythingLLM | MIT | v1.19.0, 2026-10-26 | Built-in llama.cpp, Ollama, OpenAI-compatible | GitHub, GitLab, Gitea repos, Confluence, website, Obsidian | Admin, manager, default roles per workspace (Docker only); "simple SSO passthrough" | Yes | Docker, desktop app | Active; telemetry on by default |
| Open WebUI | Custom BSD-3 plus branding clause above 50 users | v0.13.3, 2026-11-08 | Ollama, OpenAI-compatible | 44 connectors through the separate oikb tool (Git, Confluence, Jira, S3) | Groups with knowledge-base access, OIDC, LDAP, SCIM | Yes | Docker, Helm | Very active |
| Kotaemon | Apache-2.0 | v0.12.0, 2026-05-31 | Ollama, llama-cpp-python | File upload | Multi-user, private or public collections | Yes, with PDF highlight | Docker | Slow, last main commit 2026-05-30 |
| R2R | MIT | v3.6.5, 2025-06-06 | Yes | Not checked | Not checked | Yes | Compose | Last commit 2025-11-07, treat as dormant |
| Dify | Modified Apache-2.0 (no multi-tenant SaaS, keep logo) | 1.19.0, 2026-11-04 | OpenAI-compatible and many providers | Plugins: Notion, Confluence, GitHub, OneDrive, Google Drive, S3 | CE: single workspace; SSO and multi-workspace RBAC are Enterprise | Yes | Compose | Very active |
| WeKnora (Tencent) | MIT plus third-party licenses | v0.9.0, 2026-10-29 | Ollama and 29 vendors | Confluence, GitLab, Notion, Seafile, RSS, Feishu | Multi-workspace RBAC, 4 roles, audit log | Yes | Compose, Kubernetes | Active |
| LightRAG | MIT | v1.5.8rc1, 2026-11-15 | OpenAI-compatible, Ollama-compatible API | Upload only | API key or accounts; auth off by default | Yes | Compose | Very active |
| MS GraphRAG | MIT | v3.3.0, 2026-10-08 | Through LiteLLM; local models unreliable | None (library) | None | Not applicable | Library | Active, "not an officially supported Microsoft offering" |
| Haystack | Apache-2.0 | v3.5.0, 2026-11-10 | Ollama and vLLM integrations | Build your own | Build your own | Build your own | Library, Hayhooks for serving | Very active |
| LlamaIndex | MIT | v0.14.26, 2026-11-13 | Yes | LlamaHub readers | Build your own | Build your own | Library | Very active |
| LangChain | MIT | langchain-core 1.8.1, 2026-11-15 | Yes | Loaders | Build your own | Build your own | Library | Very active |

### Notes per stack

**PrivateGPT (zylon-ai).** Version 1.0 (2026-06-03) merged Zylon's private fork back into the open repository and rebuilt the project as an API that follows the Claude Messages API ([v1.0.0 notes](https://github.com/zylon-ai/private-gpt/releases/tag/v1.0.0)). It "does not run models itself" and needs an external OpenAI-compatible server. The README lists token auth as supported and "OAuth / organizations" as not supported. LDAP/AD and RBAC belong to the commercial Zylon product ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). The workbench UI at `/ui` is "a demonstrator, not the core product". It fits as a backend for a custom UI, not as a multi-user project memory.

**Onyx (formerly Danswer).** Onyx is a search-first application with chat, built for company knowledge. In the Community Edition, a single OIDC or SAML provider works. Multiple SSO providers, permission sync and user groups need Business, and SCIM and custom roles need Enterprise ([editions](https://docs.onyx.app/deployment/miscellaneous/enterprise_edition)). Permission sync covers GitHub, Confluence, Jira and SharePoint with certificate auth, among others ([connector overview](https://docs.onyx.app/admins/connectors/overview)). Onyx Standard uses OpenSearch and needs at least 4 vCPU, 10 GB RAM and 32 GB disk plus about 2.5 times the indexed data ([resourcing](https://docs.onyx.app/deployment/getting_started/resourcing)). Set `ONYX_AIRGAPPED=true` and `DISABLE_TELEMETRY=true` because self-hosted instances send anonymous telemetry by default ([FAQ](https://docs.onyx.app/security/self_hosted/faq)). Ollama has its own provider page ([Ollama](https://docs.onyx.app/admins/ai_models/ollama)). The connector overview does not say whether the Git connectors index code files or only issues and pull requests (unverified).

**RAGFlow.** RAGFlow focuses on deep document parsing (layout, OCR, tables) and template-based chunking, with grounded citations. The recommended start is 4 CPU cores, 16 GB RAM and 50 GB disk. In the 1.0 open-source release, the DeepDoc parser runs on CPU ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). The connector set is broad (see the [syncer source](https://github.com/infiniflow/ragflow/tree/main/internal/syncer/connector)). In the source, the GitHub connector "reads GitHub issues and pull requests", while the GitLab connector reads "merge requests, issues, and code files". "Text & Code" files get no extra parser controls ([parser docs](https://github.com/infiniflow/ragflow/blob/main/docs/guides/agent/ingestion_pipeline/configure_parser_component.md)). Team members who join a shared dataset can edit and delete its files, so read-only sharing is not available ([sharing docs](https://github.com/infiniflow/ragflow/blob/main/docs/guides/team/resource_sharing/resource_sharing_and_permissions.md)).

**AnythingLLM.** AnythingLLM is the easiest single-box setup. It has a built-in local LLM engine, LanceDB by default, and workspaces with three roles ([security docs](https://docs.anythingllm.com/features/security-and-access); [README](https://github.com/Mintplex-Labs/anything-llm)). It has repository loaders for GitHub, GitLab and Gitea, plus Confluence and DrupalWiki ([collector extensions](https://github.com/Mintplex-Labs/anything-llm/tree/master/collector/utils/extensions)). Telemetry goes to PostHog by default and needs `DISABLE_TELEMETRY=true`. Even then, the app contacts `cdn.anythingllm.com` for model downloads (README, "Telemetry & Privacy"). SSO is a passthrough mechanism (`SIMPLE_SSO_ENABLED`), not OIDC ([.env.example](https://github.com/Mintplex-Labs/anything-llm/blob/master/docker/.env.example)).

**Open WebUI.** Open WebUI is a strong chat front end. Its knowledge bases can be shared with groups, and groups can sync from OAuth or LDAP. Permissions are additive, with no deny rules ([groups](https://github.com/open-webui/docs/blob/main/docs/features/authentication-access/rbac/groups.md)). SCIM 2.0 is configured by environment variables ([SCIM](https://github.com/open-webui/docs/blob/main/docs/features/authentication-access/auth/scim.mdx)). Retrieval is basic: markdown-header splitting plus character or token splitters, and it can query external Qdrant, Milvus or pgvector collections ([RAG docs](https://github.com/open-webui/docs/blob/main/docs/features/chat-conversations/rag/index.md)). Connectors come from the separate MIT tool [oikb](https://github.com/open-webui/oikb), released 2026 (v0.6.0, 2026-10-21). It does incremental SHA-256 sync from 44 sources and "never bypasses Knowledge Base access control" ([oikb docs](https://github.com/open-webui/docs/blob/main/docs/ecosystem/knowledge-base-sync/index.md)). The license forbids changing the branding for deployments above 50 users without an enterprise license ([LICENSE](https://github.com/open-webui/open-webui/blob/main/LICENSE)).

**Kotaemon.** Kotaemon is a Gradio document-QA UI with good citation display (in-browser PDF highlights) and an example GraphRAG pipeline ([README](https://github.com/Cinnamon/kotaemon)). It has no source connectors and commit activity is low. It suits demos, not the reference.

**R2R.** The last release was 2025-06-06 and the last commit was 2025-11-07 ([repo](https://github.com/SciPhi-AI/R2R)). Exclude it.

**Dify.** Dify is an LLM app builder with knowledge pipelines. Data source plugins arrived in 1.9.0 and cover Notion, Confluence, GitHub, OneDrive and others ([datasource plugin guide](https://docs.dify.ai/en/develop-plugin/dev-guides-and-walkthroughs/datasource-plugin)). The license bans multi-tenant operation and logo removal ([LICENSE](https://github.com/langgenius/dify/blob/main/LICENSE)). The Community Edition is single-workspace, and SSO is an Enterprise feature ([pricing](https://dify.ai/pricing/dify-enterprise)).

**WeKnora (Tencent), added candidate.** WeKnora is MIT licensed. It has multi-workspace RBAC with an audit log, auto-sync from Confluence and GitLab, Ollama support, Neo4j for knowledge graphs, and Langfuse tracing ([README](https://github.com/Tencent/WeKnora)). Many of its connectors are China-specific (Feishu, DingTalk, Yuque). Watch it, but it is a weaker fit for EU clients.

**LightRAG and Microsoft GraphRAG.** LightRAG builds an entity-relation graph next to vector search. It supports reranking (`bge-reranker-v2-m3` recommended), document deletion with graph regeneration, PostgreSQL or Neo4j storage, and a WebUI. Authentication is off by default and the server binds to 0.0.0.0. The embedding model cannot change after indexing without re-embedding everything ([README](https://github.com/HKUDS/LightRAG)). GraphRAG warns that indexing "can be an expensive operation" and calls itself a demonstration ([README](https://github.com/microsoft/graphrag)). Its docs report frequent malformed JSON from proxied local models and say it is mainly tested with GPT-4-class models ([model config](https://microsoft.github.io/graphrag/config/models/)). Both are retrieval engines to add behind an application, not applications for end users.

**Haystack, LlamaIndex, LangChain.** These are frameworks. They fit a custom pipeline (for example, a code-aware ingestion job that writes into the same vector store). They do not fit a reference installation that consultants use directly. Haystack 3.0 (2026-07-20) focuses on agents with hooks ([release](https://github.com/deepset-ai/haystack/releases/tag/v3.0.0)).

## 2. Code as a source

How the stacks handle code:

- No end-user stack in the table does AST-aware chunking of code. RAGFlow treats code as text, Open WebUI splits by characters or tokens, and Onyx, AnythingLLM and RAGFlow pull repositories through connectors (see the notes above).
- Among the frameworks, LlamaIndex `CodeSplitter` uses tree-sitter grammars ([docs](https://developers.llamaindex.ai/python/examples/node_parsers/code_splitter_chunking/)). Haystack `PythonCodeSplitter` uses Python's `ast` module and handles Python only ([docs](https://docs.haystack.deepset.ai/docs/pythoncodesplitter)). LangChain splits code on language-specific separators, not on a syntax tree. Its separator list includes COBOL, which matters for legacy modernization ([docs](https://docs.langchain.com/oss/python/integrations/splitters/code_splitter)).

Dedicated code retrieval that runs on-prem:

| Tool | License | What it does |
|---|---|---|
| [Sourcebot](https://github.com/sourcebot-dev/sourcebot) | MIT | Self-hosted code search across GitHub, GitLab, Bitbucket and Azure DevOps. Its MCP server offers search, file reads and symbol definitions and references. "Ask" uses a configurable LLM. v6.1.0, 2026-11-10 |
| [code-graph-rag](https://github.com/vitali87/code-graph-rag) | MIT | Tree-sitter parser, optional compiler frontends, graph in embedded SQLite or Memgraph/Neo4j, natural-language queries |
| [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) | MIT | Local MCP server that builds a code, docs and infrastructure graph. Indexing needs no hosted service |
| [CodeGraphContext](https://github.com/CodeGraphContext/CodeGraphContext) | MIT | Tree-sitter and SCIP indexers into a graph database, MCP server |
| [Serena](https://github.com/oraios/serena) | GPL-3.0 (SolidLSP part MIT) | Language-server-based symbol retrieval for agents |
| [GitNexus](https://github.com/abhigyanpatwari/GitNexus) | PolyForm Noncommercial | Code graph engine. Not usable in paid consulting work |
| [tree-sitter](https://github.com/tree-sitter/tree-sitter) | MIT | Parser library under most of the above |

How to combine code retrieval with a document RAG:

1. Keep two indexes. The document RAG indexes Markdown docs, ADRs, README files, issues and pull requests through its connectors. The code index (Sourcebot or a code graph) handles source files.
2. Connect the code index to the chat layer as an MCP tool. Onyx, RAGFlow, Open WebUI and PrivateGPT all have MCP support in their code or docs (Onyx MCP test compose files in [deployment](https://github.com/onyx-dot-app/onyx/tree/main/deployment/docker_compose); RAGFlow `internal/mcp`; PrivateGPT README). The LLM then decides whether to search documents, code or both. Check per stack whether MCP tools can run against a local model with reliable tool calling.
3. Link the two indexes through shared metadata: repository, path, commit and ADR ID. For example, ADR files sit in the repository and are indexed by both.

## 3. Decision records and workshop results

The aim is answers that respect time and status, for example "what is the current decision on X" or "which decision replaced Y and why".

**Metadata first.** MADR front matter has `status` (including "superseded by ADR-0123"), `date`, `decision-makers`, `consulted` and `informed` ([MADR template](https://github.com/adr/madr/blob/develop/template/adr-template.md)). Parse these fields at ingestion and store them as chunk metadata. Then filter or rerank at query time, for example by excluding `superseded` unless the question asks for history. Backstage shows the pattern: ADR013 is marked "[superseded]" and points to ADR014 ([ADR013](https://github.com/backstage/backstage/blob/master/docs/architecture-decisions/adr013-use-node-fetch.md)). Workshop minutes need the same minimum fields: date, workshop, participants (personal data), decisions and open points. This is cheap and works in every stack that supports metadata filters. Support for metadata filters in the end-user UIs of Onyx and RAGFlow was not verified in this pass.

**Knowledge graphs.** LightRAG and GraphRAG extract entities and relations but have no time model. RAGFlow 1.0 can compile "Timelines" and "Wikis" from datasets ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). This is relevant for workshop series, but I have not tested its quality.

**Temporal memory layers.** All four below are open source and self-hostable:

| Project | License | Release | Notes |
|---|---|---|---|
| [Graphiti](https://github.com/getzep/graphiti) (Zep) | Apache-2.0 | v0.31.2, 2026-10-20 | Bi-temporal facts with validity windows. Superseded facts are "invalidated, not deleted", and every fact traces back to its source "episode". Needs Neo4j (GPL-3.0), FalkorDB (SSPL) or Neptune. Local LLMs through OpenAI-compatible endpoints. The hosted Zep platform is separate and commercial. Paper: [arXiv 2501.13956](https://arxiv.org/abs/2501.13956) |
| [Cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | v1.6.8, 2026-11-12 | Knowledge-graph memory from documents, code and conversations. Can run with no LLM, using GLiNER plus local embeddings |
| [mem0](https://github.com/mem0ai/mem0) | Apache-2.0 | ts-v3.3.1, 2026-09-25 | Agent and user memory. The current extractor is "ADD-only": memories accumulate and nothing is overwritten. Self-hosted server has auth on by default |
| [Letta](https://github.com/letta-ai/letta) | Apache-2.0 | 0.16.8, 2026-05-14 | Agent framework with memory blocks. Built for agent state, not a document corpus |

Assessment: Graphiti's model fits decision histories best, because a superseding decision invalidates the older fact and keeps the history. It adds a graph database and LLM extraction cost to every ingestion. Its "invalidate, not delete" behavior also needs a hard-delete path for GDPR erasure (section 7). For the reference installation, use metadata plus filters and keep Graphiti as an optional extension.

## 4. Local models for German and English

### Embedding models

| Model | License | Notes from the model card |
|---|---|---|
| [BAAI/bge-m3](https://huggingface.co/BAAI/bge-m3) | MIT | Dense, sparse and multi-vector output from one model, up to 8192 tokens, multilingual |
| [Qwen3-Embedding 0.6B/4B/8B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B) | Apache-2.0 | 100+ languages including programming languages, 32k context, instruction-aware, flexible dimensions. The 8B model ranked first on MTEB multilingual on 2025-06-05 |
| [multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct) | MIT | ~100 languages, inputs truncated at 512 tokens |
| [nomic-embed-text-v2-moe](https://huggingface.co/nomic-ai/nomic-embed-text-v2-moe) | Apache-2.0 | Multilingual, Matryoshka dimensions from 768 down to 256 |
| [granite-embedding-278m-multilingual](https://huggingface.co/ibm-granite/granite-embedding-278m-multilingual) | Apache-2.0 | Small multilingual option |
| [EmbeddingGemma-300m](https://huggingface.co/google/embeddinggemma-300m) | Gemma terms | Separate license terms to review |
| [jina-embeddings-v3](https://huggingface.co/jinaai/jina-embeddings-v3), [v5-text-small](https://huggingface.co/jinaai/jina-embeddings-v5-text-small) | CC BY-NC 4.0 | Non-commercial, so not usable for client work without a commercial license |

### Rerankers

- [bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3) (Apache-2.0, multilingual) is the reranker LightRAG recommends.
- [Qwen3-Reranker 0.6B/4B](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B) is Apache-2.0.
- The Jina rerankers v2 and v3 are CC BY-NC 4.0 ([v3](https://huggingface.co/jinaai/jina-reranker-v3)).

### LLMs for one GPU or Apple Silicon

| Model | License | Size and fit |
|---|---|---|
| [gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b) | Apache-2.0 | Runs "within 16GB of memory" (MXFP4). [gpt-oss-120b](https://huggingface.co/openai/gpt-oss-120b) fits one 80 GB GPU |
| [Qwen3.6-35B-A3B](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) | Apache-2.0 | MoE with 35B total and 3B active parameters, 262k context. PrivateGPT's README lists the Qwen3.5 35B Ollama build at ~24 GB |
| [Qwen3.6-27B](https://huggingface.co/Qwen/Qwen3.6-27B) | Apache-2.0 | Dense model |
| [Mistral-Small-3.2-24B](https://huggingface.co/mistralai/Mistral-Small-3.2-24B-Instruct-2506) | Apache-2.0 | ~55 GB in bf16, so it needs quantization on a single 24-48 GB card |
| [Ministral-3-14B](https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512) | Apache-2.0 | Smaller EU-vendor option |
| [Gemma 4 31B](https://huggingface.co/google/gemma-4-31B-it) | Apache-2.0 | 140+ languages, 256k context |
| [EuroLLM-22B-Instruct-2512](https://huggingface.co/utter-project/EuroLLM-22B-Instruct-2512) | Apache-2.0 | Trained for all EU languages, 32k context |
| [Teuken-7B-instruct-commercial-v0.4](https://huggingface.co/openGPT-X/Teuken-7B-instruct-commercial-v0.4) | Apache-2.0 | German-led project. The newer [v0.6](https://huggingface.co/openGPT-X/Teuken-7B-instruct-v0.6) is CC BY-NC 4.0 |
| Llama 4 | Llama 4 Community License | The use policy withholds the license grant for multimodal Llama 4 models from EU-domiciled individuals and companies ([policy](https://github.com/meta-llama/llama-models/blob/main/models/llama4/USE_POLICY.md)). Avoid it |

Serving options:

- [Ollama](https://github.com/ollama/ollama) (MIT, v0.40.16) is simple on a Mac or a single GPU. Its default context is 4096 tokens on GPUs with less than 24 GiB, which Open WebUI's docs flag as a common cause of bad RAG answers ([RAG docs](https://github.com/open-webui/docs/blob/main/docs/features/chat-conversations/rag/index.md)). Raise it.
- [llama.cpp](https://github.com/ggml-org/llama.cpp) (MIT, v1.0.0, 2026-11-13) is a lower-level option.
- [vLLM](https://github.com/vllm-project/vllm) (Apache-2.0, v0.34.0) is the choice for concurrent users on NVIDIA.

None of the model cards I checked reports German-specific retrieval or QA scores. Run the evaluation in section 5 on German questions before you fix the model.

## 5. Evaluation

Metrics that matter:

| Area | Metric | Tool support |
|---|---|---|
| Retrieval | Context recall, context precision, hit rate at k | [Ragas](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/) (context precision, recall, entities recall, noise sensitivity); [DeepEval](https://deepeval.com/docs/metrics-introduction) (contextual precision, recall, relevancy) |
| Generation | Faithfulness or groundedness, answer relevancy | Ragas, DeepEval, [TruLens RAG triad](https://www.trulens.org/getting_started/core_concepts/rag_triad/) (context relevance, groundedness, answer relevance) |
| Citations | Citation precision and recall: does each cited source support the claim, and is each claim cited | Defined in ALCE ([Gao et al. 2023](https://arxiv.org/abs/2305.14627)). Not a built-in metric in Ragas or DeepEval as far as checked, so implement it as a custom metric |
| Security | ACL leakage: user B never gets chunks from project A | Custom test |
| Erasure | After deleting a source document, no answer or citation still contains it | Custom test |

Tools:

- [Ragas](https://github.com/explodinggradients/ragas) (Apache-2.0, v0.4.4, 2026-11-14) generates test sets from documents through a knowledge graph built with an LLM and embeddings ([testset generation](https://docs.ragas.io/en/stable/getstarted/rag_testset_generation/)).
- [DeepEval](https://github.com/confident-ai/deepeval) (Apache-2.0) supports Ollama judges with `deepeval set-ollama`, but warns that custom models may not produce valid JSON. Its Synthesizer builds "goldens" from .txt, .docx, .pdf and .md files ([synthesizer](https://deepeval.com/docs/synthesizer-introduction)).
- [TruLens](https://github.com/truera/trulens) is MIT.
- For tracing, [Langfuse](https://github.com/langfuse/langfuse) is MIT except its `ee` folders. [Arize Phoenix](https://github.com/Arize-ai/phoenix) is Elastic License 2.0.

Suggested procedure:

1. Write 50 to 100 gold questions by hand, in German and English, split into categories: fact lookup in docs, decision rationale ("why X"), current status ("is ADR-013 still valid"), code location ("where is X implemented"), and cross-source questions.
2. Add synthetic questions from Ragas or DeepEval.
3. Run the evaluation with a local judge model, and check a sample of judge verdicts by hand.
4. Record the score per category for each model and chunking setting.

## 6. Realistic test datasets

| Dataset | License | Why it fits |
|---|---|---|
| [Backstage](https://github.com/backstage/backstage) | Apache-2.0 | Large TypeScript monorepo with docs and 15 ADRs in `docs/architecture-decisions`. ADR013 is superseded by ADR014, which tests status handling |
| [GOV.UK govuk-infrastructure](https://github.com/alphagov/govuk-infrastructure) | MIT | Terraform and Kubernetes code with ADRs in `docs/architecture/decisions` (26 files including a template). They document a platform migration to AWS EKS, which is a real modernization story |
| [Apache Kafka](https://github.com/apache/kafka) plus KIPs | Apache-2.0 | KIPs are decision records on a public Confluence instance (cwiki.apache.org), so they test the Confluence connector against real Confluence. [KIP-500](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum) (replace ZooKeeper, status "Accepted") is a multi-year modernization and links its dev mailing-list thread on lists.apache.org |
| [spring-petclinic-microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) | Apache-2.0 | Small, known monolith-to-microservices example. Useful for code questions, thin on docs |
| [AMI Meeting Corpus](https://groups.inf.ed.ac.uk/ami/corpus/) | CC BY 4.0 | 100 hours of meetings, about two thirds role-played design-team projects from kick-off to completion. A realistic source for workshop minutes |

Synthetic option: take one public repository (GOV.UK or Backstage) and use a local LLM to write workshop minutes and decision logs that refer to its real ADRs, issues and commits. Insert fictitious personal data (names, roles, one sensitive remark) on purpose. This gives known answers for evaluation and known targets for erasure tests. Mailing-list archives contain real names and email addresses of third parties. They are public, but GDPR still applies, so keep them out of shared demos or pseudonymize them.

## 7. Privacy and GDPR

The main sources are the German DSK guidance on RAG systems ([Orientierungshilfe RAG, v1.0, Oct 2025](https://datenschutzkonferenz-online.de/media/oh/DSK_OH_RAG.pdf)), the DSK guidance on technical and organizational measures for AI systems ([June 2025](https://www.datenschutzkonferenz-online.de/media/oh/DSK-OH_KI-Systeme.pdf)), [EDPB Opinion 28/2024](https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en) and the EDPB expert report on LLM privacy risks ([April 2025](https://edpb.europa.eu/system/files/2025-04/ai-privacy-risks-and-mitigations-in-llms.pdf)). The expert report is by an external author and not an official EDPB position.

**Legal basis.** Embedding personal data from reference documents and storing it in a vector database is processing that needs its own legal basis (DSK RAG, p. 13-14). Workshop minutes with names, roles and opinions are the main personal data in this corpus. Decide what goes in. The DSK guidance says to add only the documents that are needed (p. 12-13). Check whether a DPIA under [Art. 35 GDPR](https://eur-lex.europa.eu/eli/reg/2016/679/oj) is required.

**Access control at retrieval time.** The DSK states that the rights and roles concept applies to areas of the vector database and the reference documents, and that the LLM itself cannot control who sees what (p. 11). For purpose limitation, it recommends client or functional separation in the vector database (p. 12). In practice, this means one of the following:

- A separate instance or dataset per client project.
- ACL filters applied inside the retrieval query, as in Onyx ([access controls](https://docs.onyx.app/security/architecture/access_controls)).

Never filter after the LLM has seen the chunks.

**Erasure (Art. 17).** The DSK notes that vector database entries are directly addressable and deletable, and that normal retention-period deletion works for the vector database and source documents (p. 13). Deletion must also reach the following places:

1. The source system. The connector must sync deletions, which oikb does with explicit deletion handling and RAGFlow with checkpointed syncs.
2. The chunk store and the vector index.
3. Derived artifacts: graph entities and summaries (LightRAG regenerates the graph on delete; Graphiti only invalidates by default), RAGFlow compiled wikis and timelines, and LLM response caches.
4. Chat histories and logs that contain retrieved snippets.
5. Evaluation datasets.
6. Backups, under the backup retention period.

Test erasure as part of the evaluation (section 5).

**Logging.** The EDPB report lists "insecure logging or caching" of queries and retrieved documents as a common RAG risk (p. 34) and recommends limited log retention by default. Query logs show who searched for what. In Germany, a system that can monitor employee behavior is subject to works council co-determination under [§ 87(1) No. 6 BetrVG](https://www.gesetze-im-internet.de/betrvg/__87.html). Set retention periods, pseudonymize logs where you can, and restrict admin access to chat histories.

**Outbound connections.** "On-premise" requires turning off default telemetry: Onyx `DISABLE_TELEMETRY` and `ONYX_AIRGAPPED`, AnythingLLM `DISABLE_TELEMETRY`. Pin model downloads to an internal mirror. Check model licenses as well (Llama 4 EU clause, CC BY-NC models in section 4).

**Other points.** The DSK names membership inference and data poisoning attacks on the RAG subsystem (p. 12), so accept only documents from known sources (p. 11). Under the [AI Act Art. 4](https://eur-lex.europa.eu/eli/reg/2024/1689/oj), deployers must ensure AI literacy of staff. This obligation has applied since 2 February 2025.

## Sources

Repositories and releases (GitHub API, checked 2026-11-17):

- c4 GenAI Suite: https://github.com/codecentric/c4-genai-suite (archived; release v10.0.4)
- PrivateGPT: https://github.com/zylon-ai/private-gpt, https://github.com/zylon-ai/private-gpt/releases/tag/v1.0.0
- Onyx: https://github.com/onyx-dot-app/onyx, https://github.com/onyx-dot-app/onyx/blob/main/LICENSE
- RAGFlow: https://github.com/infiniflow/ragflow, https://github.com/infiniflow/ragflow/releases/tag/v1.0.0
- AnythingLLM: https://github.com/Mintplex-Labs/anything-llm
- Open WebUI: https://github.com/open-webui/open-webui, https://github.com/open-webui/oikb, https://github.com/open-webui/docs
- Kotaemon: https://github.com/Cinnamon/kotaemon
- R2R: https://github.com/SciPhi-AI/R2R
- Dify: https://github.com/langgenius/dify
- WeKnora: https://github.com/Tencent/WeKnora
- LightRAG: https://github.com/HKUDS/LightRAG
- GraphRAG: https://github.com/microsoft/graphrag
- Haystack: https://github.com/deepset-ai/haystack
- LlamaIndex: https://github.com/run-llama/llama_index
- LangChain: https://github.com/langchain-ai/langchain
- Code tools: https://github.com/sourcebot-dev/sourcebot, https://github.com/vitali87/code-graph-rag, https://github.com/DeusData/codebase-memory-mcp, https://github.com/CodeGraphContext/CodeGraphContext, https://github.com/oraios/serena, https://github.com/abhigyanpatwari/GitNexus, https://github.com/tree-sitter/tree-sitter
- Memory layers: https://github.com/getzep/graphiti, https://github.com/topoteretes/cognee, https://github.com/mem0ai/mem0, https://github.com/letta-ai/letta
- Evaluation: https://github.com/explodinggradients/ragas, https://github.com/confident-ai/deepeval, https://github.com/truera/trulens, https://github.com/langfuse/langfuse, https://github.com/Arize-ai/phoenix
- Inference: https://github.com/ollama/ollama, https://github.com/vllm-project/vllm, https://github.com/ggml-org/llama.cpp
- ADR format: https://github.com/adr/madr

Documentation:

- Onyx: https://docs.onyx.app/deployment/miscellaneous/enterprise_edition, https://docs.onyx.app/security/architecture/access_controls, https://docs.onyx.app/admins/connectors/overview, https://docs.onyx.app/deployment/getting_started/resourcing, https://docs.onyx.app/security/self_hosted/faq, https://docs.onyx.app/admins/ai_models/ollama
- AnythingLLM: https://docs.anythingllm.com/features/security-and-access
- Dify: https://docs.dify.ai/en/develop-plugin/dev-guides-and-walkthroughs/datasource-plugin, https://dify.ai/pricing/dify-enterprise
- GraphRAG: https://microsoft.github.io/graphrag/config/models/
- LlamaIndex: https://developers.llamaindex.ai/python/examples/node_parsers/code_splitter_chunking/
- Haystack: https://docs.haystack.deepset.ai/docs/pythoncodesplitter
- LangChain: https://docs.langchain.com/oss/python/integrations/splitters/code_splitter
- Ragas: https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/, https://docs.ragas.io/en/stable/getstarted/rag_testset_generation/
- DeepEval: https://deepeval.com/docs/metrics-introduction, https://deepeval.com/docs/synthesizer-introduction
- TruLens: https://www.trulens.org/getting_started/core_concepts/rag_triad/
- C4 model: https://c4model.com/

Model cards (Hugging Face; licenses from the model API):

- Embeddings: BAAI/bge-m3, Qwen/Qwen3-Embedding-0.6B/4B/8B, intfloat/multilingual-e5-large-instruct, nomic-ai/nomic-embed-text-v2-moe, ibm-granite/granite-embedding-278m-multilingual, google/embeddinggemma-300m, jinaai/jina-embeddings-v3, jinaai/jina-embeddings-v5-text-small
- Rerankers: BAAI/bge-reranker-v2-m3, Qwen/Qwen3-Reranker-0.6B, jinaai/jina-reranker-v3
- LLMs: openai/gpt-oss-20b, openai/gpt-oss-120b, Qwen/Qwen3.6-35B-A3B, Qwen/Qwen3.6-27B, mistralai/Mistral-Small-3.2-24B-Instruct-2506, mistralai/Ministral-3-14B-Instruct-2512, google/gemma-4-31B-it, utter-project/EuroLLM-22B-Instruct-2512, openGPT-X/Teuken-7B-instruct-commercial-v0.4
- Llama 4 use policy: https://github.com/meta-llama/llama-models/blob/main/models/llama4/USE_POLICY.md

Papers:

- Rasmussen et al., Zep: A Temporal Knowledge Graph Architecture: https://arxiv.org/abs/2501.13956
- Gao et al., Enabling Large Language Models to Generate Text with Citations (ALCE): https://arxiv.org/abs/2305.14627

Regulators and law:

- DSK, Orientierungshilfe zu datenschutzrechtlichen Besonderheiten generativer KI-Systeme mit RAG-Methode, v1.0, October 2025: https://datenschutzkonferenz-online.de/media/oh/DSK_OH_RAG.pdf
- DSK, Orientierungshilfe zu empfohlenen technischen und organisatorischen Maßnahmen bei der Entwicklung und beim Betrieb von KI-Systemen, v1.0, June 2025: https://www.datenschutzkonferenz-online.de/media/oh/DSK-OH_KI-Systeme.pdf
- EDPB Opinion 28/2024 on AI models: https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en
- EDPB Support Pool of Experts, AI Privacy Risks & Mitigations: LLMs, April 2025: https://edpb.europa.eu/system/files/2025-04/ai-privacy-risks-and-mitigations-in-llms.pdf
- GDPR: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- AI Act: https://eur-lex.europa.eu/eli/reg/2024/1689/oj
- BetrVG § 87: https://www.gesetze-im-internet.de/betrvg/__87.html

Datasets:

- Backstage ADRs: https://github.com/backstage/backstage/tree/master/docs/architecture-decisions
- GOV.UK ADRs: https://github.com/alphagov/govuk-infrastructure/tree/main/docs/architecture/decisions
- Apache Kafka KIP-500: https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum
- spring-petclinic-microservices: https://github.com/spring-petclinic/spring-petclinic-microservices
- AMI Meeting Corpus: https://groups.inf.ed.ac.uk/ami/corpus/
