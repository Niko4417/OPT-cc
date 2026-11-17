# Candidate stacks scored against the local-first rules

Date: 2026-11-17. Ticket: [#2](https://github.com/Niko4417/OPT-cc/issues/2). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

Which open-source stacks can be the base app of the project memory, and how does each one score against seven rules: native install without Docker Desktop, fits next to the models on a 64 GB Mac, talks to one OpenAI-compatible gateway (LiteLLM), ingests GitHub repo files plus issues and pull requests, answers with citations, exports and imports one project memory, and enforces a model policy per project memory? Licenses must allow free commercial use. Keiko gets no bonus.

The scope was widened during the ticket. The survey covers actively maintained open-source RAG systems beyond the background report, including retrieval baselines that report scores on public benchmarks. Keiko is scored twice: as it is on its default branch today, and with the work that is planned or in flight. The output ends with two or three candidates to measure head to head on the gold question set. Measured answer quality decides the stack, not this table.

Ticket [#5](https://github.com/Niko4417/OPT-cc/issues/5) found that no candidate's own GitHub connector covers code, issues, pull requests and comments, and planned a separate ingestion job ([github-ingestion.md](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md)). So this note adds rule 4b: how easily the app accepts externally produced chunks with citation URLs.

## Answer

No candidate passes all eight rules, and no candidate's public benchmark numbers say anything about this task.

- Open WebUI leads with 12 of 16 points, Keiko (dev, 1.5.1) follows with 11, and LightRAG and Dify have 10.
- Open WebUI is the only lightweight candidate that already reads externally produced chunks with a URL field: an external knowledge base over a Qdrant, Milvus or pgvector collection maps content, source and URL fields into citations. That fits the ingestion job from ticket #5 without app changes.
- Keiko is the only candidate that enforces a per-project-memory model policy in code, and it already runs against LiteLLM with the lab model pair. It cannot export or import a Knowledge Pod, it does not index GitHub issues or pull requests, and its policy blocks operations instead of model locations. Its connector pods already map each synced item to a web URL for citations, but only for Jira and Confluence.
- The planned Keiko work (epic #4429 and issue #4338) fixes a crash, a slow repeat search and a wrong-file bug in connected-folder search, and lets the person pick the embedding model. It does not change any rule score. It does change answer quality, so the measured build must include it.
- Onyx has a document ingestion API with links per section, but its Standard mode needs at least 10 GB RAM in containers, which does not fit next to the models. RAGFlow needs 16 GB and has no macOS build of its Go backend.
- The projects that report scores on public RAG benchmarks are research libraries or agents built for another task: HippoRAG 2, FlashRAG, PageIndex, PaperQA2, local-deep-research and Khoj. None of the end-user apps, Keiko included, reports such scores. Keiko's own retrieval gates run on synthetic fixtures.

Shortlist to measure head to head on the gold question set:

1. Keiko from `dev`, with the #4456 and #4458 fixes once they land. It is hybrid retrieval with reranking, an entailment check and the only working policy hook.
2. Open WebUI with hybrid search (BM25 plus vectors plus reranker), fed by the ticket #5 ingestion job. It is the widely used plain-RAG baseline that installs with pip and talks to LiteLLM.
3. LightRAG. It is the graph-based retrieval contrast that also installs with pip and talks to LiteLLM.

## Method

- Rules and scale. The seven rules from the ticket plus rule 4b from ticket #5. Each rule scores 2 (meets the rule as shipped), 1 (meets part of it, or meets it with small glue) or 0 (missing, or blocked). The license is a gate, not a score.
- Sources. Source code and docs of each candidate on 2026-11-17, read through the GitHub API and the vendors' docs sites. Keiko was read from `origin/dev` at commit `d72d6db6f` (2026-11-17, release 1.5.1). The LiteLLM lab script exists only on the Keiko for Quality branch, read at commit `b5fe2a8`. Paths written as `Keiko:path` refer to `dev` unless marked.
- Starting list. The background report `docs/research/rag-stack-landscape.md` on branch `research/rag-stack-landscape` gave the first eight candidates. The wider survey adds 30 more repositories. I re-checked every claim I reuse from the background report.
- Activity test. A project counts as active if its repository had a push in the last three months and a release in 2026. The data comes from the GitHub API on 2026-11-17.
- Limits. I installed nothing and measured no memory or answer quality. Memory scores rest on vendor minimums and on process architecture. Ticket [#4](https://github.com/Niko4417/OPT-cc/issues/4) measures the model side, ticket [#5](https://github.com/Niko4417/OPT-cc/issues/5) goes deeper on GitHub ingestion, ticket [#6](https://github.com/Niko4417/OPT-cc/issues/6) defines the export bundle, and ticket [#12](https://github.com/Niko4417/OPT-cc/issues/12) runs the trial.

## Wider survey

Stars, last push and latest release come from the GitHub API on 2026-11-17. "Scored" means the candidate appears in the rule tables below.

### End-user apps

| Repository | License | Last push | Latest release | Verdict |
|---|---|---|---|---|
| [open-webui/open-webui](https://github.com/open-webui/open-webui) | Open WebUI License (BSD-3 plus branding clause) | 2026-11-17 | v0.13.4, 2026-11-17 | Scored |
| [langgenius/dify](https://github.com/langgenius/dify) | Modified Apache-2.0 | 2026-11-17 | 1.19.0, 2026-11-04 | Scored |
| [infiniflow/ragflow](https://github.com/infiniflow/ragflow) | Apache-2.0 | 2026-11-17 | v1.0.0, 2026-11-06 | Scored |
| [Mintplex-Labs/anything-llm](https://github.com/Mintplex-Labs/anything-llm) | MIT (Desktop binary proprietary) | 2026-11-17 | v1.19.0, 2026-10-26 | Scored |
| [onyx-dot-app/onyx](https://github.com/onyx-dot-app/onyx) | MIT except `ee` | 2026-11-17 | v5.1.3, 2026-11-16 | Scored |
| [zylon-ai/private-gpt](https://github.com/zylon-ai/private-gpt) | Apache-2.0 | 2026-11-17 | v1.0.1, 2026-06-18 | Scored |
| [MODSetter/SurfSense](https://github.com/MODSetter/SurfSense) | Apache-2.0, some plugins need a paid license | 2026-11-14 | v2.3.1, 2026-11-03 | Scored |
| oscharko-dev/Keiko | Apache-2.0 plus trademark policy | 2026-11-17 | v1.5.1, 2026-11-16 | Scored |
| [arc53/DocsGPT](https://github.com/arc53/DocsGPT) | MIT | 2026-11-17 | 1.1.0, 2026-10-25 | Not scored. The installer requires Docker, and its GitHub loader reads repo files through the tree and contents APIs only ([github_loader.py](https://github.com/arc53/DocsGPT/blob/main/docsgpt/parser/remote/github_loader.py)). It would score like Open WebUI with a worse install |
| [Tencent/WeKnora](https://github.com/Tencent/WeKnora) | MIT plus third-party licenses | 2026-11-16 | v0.9.0, 2026-10-29 | Not scored. Docker Compose or Kubernetes, connectors aimed at Chinese services (background report) |
| [1Panel-dev/MaxKB](https://github.com/1Panel-dev/MaxKB) | GPL-3.0 | 2026-11-17 | v2.10.8-lts, 2026-10-22 | Not scored. Copyleft on any glue that modifies it, ships as containers |
| [labring/FastGPT](https://github.com/labring/FastGPT) | Apache-2.0 with extra conditions like Dify's ([LICENSE](https://github.com/labring/FastGPT/blob/main/LICENSE)) | 2026-11-17 | v4.17.4, 2026-11-05 | Not scored. Same license shape and container stack as Dify, with fewer EU-relevant connectors |
| [khoj-ai/khoj](https://github.com/khoj-ai/khoj) | AGPL-3.0 | 2026-08-02 | 2.0.0-beta.28, 2026-03-26 | Not scored. No push in 3.5 months, network copyleft |
| [morphik-org/morphik-core](https://github.com/morphik-org/morphik-core) | Business Source License 1.1, production use only under US $2,000 revenue per month ([LICENSE](https://github.com/morphik-org/morphik-core/blob/main/LICENSE)) | 2026-11-04 | none | Fails the license gate |
| [Cinnamon/kotaemon](https://github.com/Cinnamon/kotaemon) | Apache-2.0 | 2026-07-14 | v0.12.0, 2026-05-31 | Not scored. Upload only, no push in four months |
| [The-Vibe-Company/quivr](https://github.com/The-Vibe-Company/quivr) | MIT | 2026-11-16 | v2.0.3, 2026-11-16 | Not scored. Quivr 2 is an alerting and search API for incoming content with "no user interface of its own" ([README](https://github.com/The-Vibe-Company/quivr)) |
| [weaviate/Verba](https://github.com/weaviate/Verba) | BSD-3-Clause | 2026-06-08 | v2.1.3, 2025-07-14 | Archived |
| [truefoundry/cognita](https://github.com/truefoundry/cognita) | Apache-2.0 | 2026-11-17 | none | Archived |
| [h2oai/h2ogpt](https://github.com/h2oai/h2ogpt) | Apache-2.0 | 2025-10-09 | 0.2.1, 2024-06-04 | Archived |
| [SciPhi-AI/R2R](https://github.com/SciPhi-AI/R2R) | MIT | 2025-11-07 | v3.6.5, 2025-06-06 | Dormant |
| [ItzCrazyKns/Vane](https://github.com/ItzCrazyKns/Vane) (formerly Perplexica) | MIT | 2026-09-01 | v1.12.2, 2026-04-10 | Not scored. Web search answer engine, not a document memory |
| c4 GenAI Suite | Apache-2.0 | archived | v10.0.4, 2026-10-23 | Archived (background report) |

### Retrieval engines and baselines

| Repository | License | Last push | Latest release | Public benchmark it reports | Verdict |
|---|---|---|---|---|---|
| [HKUDS/LightRAG](https://github.com/HKUDS/LightRAG) | MIT | 2026-11-17 | v1.5.8rc1, 2026-11-15 | LLM-judged win rates against NaiveRAG, HyDE and GraphRAG on the UltraDomain sets ([paper, arXiv 2410.05779](https://arxiv.org/abs/2410.05779)). These are pairwise preferences, not accuracy | Scored. It has a server, a web UI and citations |
| [OSU-NLP-Group/HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG) | MIT | 2026-10-06 | v1.0.0, 2025-02-27 | NaturalQuestions, PopQA, NarrativeQA, MuSiQue, 2Wiki, HotpotQA and LV-Eval ([README](https://github.com/OSU-NLP-Group/HippoRAG); [paper, arXiv 2502.14802](https://arxiv.org/abs/2502.14802)) | Not scored. No release in 2026, a research library with no app, no citations UI and no connectors. Strongest published multi-hop numbers in this list |
| [RUC-NLPIR/FlashRAG](https://github.com/RUC-NLPIR/FlashRAG) | MIT | 2026-10-27 | v0.4.1, 2026-10-27 | 36 pre-processed benchmark datasets and 23 methods with a result table, for example F1 "close to 60" on HotpotQA for reasoning pipelines ([README](https://github.com/RUC-NLPIR/FlashRAG)) | Not scored. A research toolkit for comparing methods on Wikipedia-style corpora, not an app |
| [VectifyAI/PageIndex](https://github.com/VectifyAI/PageIndex) | MIT | 2026-11-17 | v0.7.1, 2026-11-17 | Its own Financial QA benchmark: 100 questions over 40 Form 10-K filings ([README](https://github.com/VectifyAI/PageIndex)) | Not scored. Reasoning-based retrieval over long PDFs, built around one document tree per file. It does not fit a repo of many small files |
| [Future-House/paper-qa](https://github.com/Future-House/paper-qa) | Apache-2.0 | 2026-11-16 | v2026.08.12 | LitQA2 ([README](https://github.com/Future-House/paper-qa)) | Not scored. Built for scientific papers |
| [LearningCircuit/local-deep-research](https://github.com/LearningCircuit/local-deep-research) | MIT | 2026-11-17 | v2.0.1, 2026-11-15 | About 95% on SimpleQA (n=500) with Qwen3.6-27B, using cloud web search ([README](https://github.com/LearningCircuit/local-deep-research)) | Not scored. The result measures web research, not a private corpus |
| [microsoft/graphrag](https://github.com/microsoft/graphrag) | MIT | 2026-11-16 | v3.3.0, 2026-10-08 | None in README | Not scored. Library, unreliable with local models (background report) |
| [HKUDS/RAG-Anything](https://github.com/HKUDS/RAG-Anything) | MIT | 2026-11-16 | v1.4.4, 2026-10-28 | Multimodal document benchmarks | Not scored. A multimodal layer on LightRAG |
| [Marker-Inc-Korea/AutoRAG](https://github.com/Marker-Inc-Korea/AutoRAG) | Apache-2.0 | 2026-11-17 | v2.13.13, 2026-11-17 | None. It searches pipeline settings against your own QA set | Not scored. A possible tuning tool for the trial, not a base app |
| [topoteretes/cognee](https://github.com/topoteretes/cognee), [getzep/graphiti](https://github.com/getzep/graphiti) | Apache-2.0 | 2026-11 | v1.6.8, v0.31.2 | Conversational memory benchmarks (BEAM for Cognee) | Not scored. Memory layers, see background report |
| [neuml/txtai](https://github.com/neuml/txtai), [StarTrail-org/LEANN](https://github.com/StarTrail-org/LEANN), [superlinear-ai/raglite](https://github.com/superlinear-ai/raglite) | Apache-2.0, MIT, MPL-2.0 | 2026-11 | 2026 releases | None for answer quality | Not scored. Libraries without an end-user app |
| deepset-ai/haystack, run-llama/llama_index | Apache-2.0, MIT | 2026-11 | 2026 releases | None | Not scored. Frameworks |
| [sourcebot-dev/sourcebot](https://github.com/sourcebot-dev/sourcebot) | The GitHub API still reports MIT, but `main` has no LICENSE file on 2026-11-17. Ticket #5 reports that v6 went closed source and that the last open release, v5.1.16, is FSL-1.1-ALv2 | 2026-11-10 | v6.1.0, 2026-11-10 | None | Not a candidate. The background report's code index suggestion no longer holds |

What the benchmark claims mean for this ticket:

- None of the end-user apps reports a score on a public RAG benchmark.
- The published scores use Wikipedia-style QA (HippoRAG 2, FlashRAG), long financial PDFs (PageIndex), scientific papers (PaperQA2) or open web search (local-deep-research). None of them covers code, issues, pull requests or decision records, and none covers German.
- Keiko's gates `check:retrieval-quality` and `check:grounded-retrieval-quality` report top-k recall, MRR and nDCG@k "over fixed synthetic repositories" (`Keiko:scripts/check-retrieval-quality.mjs`). They test for regressions. They cannot compare Keiko with another stack.

So the gold question set from ticket #11 is the only fair comparison.

## Keiko today and in flight

Keiko's default branch is `dev`. On 2026-11-17 it holds release 1.5.1 (commit `d72d6db6f`).

What changed between the 1.4.0 commit read first (`b5fe2a8`) and `dev`:

- The parsers were rewritten for DOCX, PPTX, XLSX, PDF layout, HTML, JSON and text structure.
- The code parser now cuts source files at top-level declarations and type members, with their leading comments. It uses indentation and line patterns, not a syntax tree (`Keiko:packages/keiko-local-knowledge/src/parsers/code-parser.ts`).
- The model-use policy, the pod store and the missing export are unchanged.

Planned or in-flight work on search, from the open issues. No pull request for any of these existed on 2026-11-17.

| Issue | Opened | What it changes | Effect on this ticket |
|---|---|---|---|
| [#4456](https://github.com/oscharko-dev/Keiko/issues/4456) | 2026-11-17 | One grounded question over a connected repository of about 10,600 files grew the server from 0.5 GB to 4.27 GB and crashed it on Windows. The issue asks to bound the allocation and to check macOS | Answer quality and rule 2. The trial corpus may be that large |
| [#4457](https://github.com/oscharko-dev/Keiko/issues/4457) | 2026-11-17 | The workspace index caches exact queries, so a differently worded question re-reads every file: 87 s per question on Windows. The fix adds a per-file layer, with a target under 10 s | Answer latency, not a rule |
| [#4458](https://github.com/oscharko-dev/Keiko/issues/4458) | 2026-11-17 | A new question in the same chat was answered only from the previous answer's file, while search ranked the right file first (score 166 against 65) | Answer quality in multi-question sessions |
| [#4434](https://github.com/oscharko-dev/Keiko/issues/4434), [#4431](https://github.com/oscharko-dev/Keiko/issues/4431) | 2026-11-16 | Skip unreadable entries instead of failing the whole search, and log discovery drops | Completeness of ingestion |
| Other children of epic [#4429](https://github.com/oscharko-dev/Keiko/issues/4429) | 2026-11-16 | Windows-specific walking, globbing and CRLF handling | Linux and Windows later, no effect on the Mac reference |
| [#4338](https://github.com/oscharko-dev/Keiko/issues/4338) | 2026-11-10 | Let the person pick the embedding model. Today Keiko sorts embedding models by cost class and config order, so a cloud model can win silently. The issue requires fail-closed binding with no silent fallback | Rule 7. It makes "embed on-device" a visible, enforced choice |
| Knowledge epics [#2559](https://github.com/oscharko-dev/Keiko/issues/2559) (graph grounding), [#2560](https://github.com/oscharko-dev/Keiko/issues/2560) (deep research), [#2558](https://github.com/oscharko-dev/Keiko/issues/2558) (shared pod curation) | 2026-07 | Roadmap, last updated in July | None yet |

Epic #4429 and its children change the connected-folder search in chat. That path searches files live per question. Knowledge Pods use a separate indexed path with lexical and vector lanes and reranking. A project memory can use either path, so the trial should state which one it measures.

## Candidates scored

| Candidate | Version checked | License | License gate |
|---|---|---|---|
| Keiko | 1.5.1, `dev` at `d72d6db6f` | Apache-2.0 (`Keiko:LICENSE`), plus a trademark policy that bars using the Keiko name or logo as the main brand of a derivative product (`Keiko:TRADEMARKS.md`) | Pass |
| Open WebUI | v0.13.4 | BSD-3 plus a branding clause. Removing the branding is allowed only up to 50 end users in 30 days, with written permission, or with an enterprise license ([LICENSE](https://github.com/open-webui/open-webui/blob/main/LICENSE)) | Pass, keep the branding |
| LightRAG | v1.5.8rc1 | MIT | Pass |
| Dify CE | 1.19.0 | Modified Apache-2.0. Commercial use is allowed. A multi-tenant service (one tenant is one workspace) and removing the logo need a commercial license. The producer "can adjust the open-source agreement" ([LICENSE](https://github.com/langgenius/dify/blob/main/LICENSE)) | Pass, with conditions |
| AnythingLLM | v1.19.0 | Repo is MIT. The Desktop binary is not: its terms limit use to "personal and / or internal business purposes" and call its source code a trade secret ([Desktop terms](https://docs.anythingllm.com/installation-desktop/terms)) | Pass for the MIT source build only |
| SurfSense | v2.3.1 | Apache-2.0. The plugins "By SurfSense" need a SurfSense license ([README](https://github.com/MODSetter/SurfSense)) | Pass without those plugins |
| Onyx CE | v5.1.3 | MIT except `ee` folders | Pass |
| RAGFlow | v1.0.0 | Apache-2.0 | Pass |
| PrivateGPT | 1.0 line | Apache-2.0 | Pass |

## Summary matrix

| Candidate | 1 Native install | 2 Fits in 64 GB | 3 One gateway | 4 Own GitHub connector | 4b External chunks with URLs | 5 Citations | 6 Export and import | 7 Model policy | Total of 16 |
|---|---|---|---|---|---|---|---|---|---|
| Open WebUI | 2 | 2 | 2 | 1 | 2 | 2 | 1 | 0 | 12 |
| Keiko, `dev` today | 2 | 2 | 2 | 1 | 1 | 2 | 0 | 1 | 11 |
| Keiko, with in-flight issues | 2 | 2 | 2 | 1 | 1 | 2 | 0 | 1 | 11 |
| LightRAG | 2 | 2 | 2 | 0 | 1 | 2 | 1 | 0 | 10 |
| Dify CE | 1 | 1 | 2 | 2 | 1 | 2 | 0 | 1 | 10 |
| AnythingLLM (MIT build) | 1 | 2 | 1 | 1 | 1 | 2 | 0 | 1 | 9 |
| Onyx CE | 1 | 0 | 2 | 1 | 2 | 2 | 0 | 0 | 8 |
| RAGFlow | 1 | 0 | 2 | 1 | 1 | 2 | 0 | 1 | 8 |
| PrivateGPT | 2 | 2 | 1 | 0 | 1 | 2 | 0 | 0 | 8 |
| SurfSense | 2 | 2 | 1 | 0 | 0 | 2 | 0 | 0 | 7 |

The in-flight Keiko work changes no score. Issue #4338 makes Keiko's rule 7 stronger inside the 1 (an explicit, fail-closed embedding choice), but the policy still does not know model locations.

The totals weight all rules equally. The rules do not cost the same to fix, so the shortlist ranks by glue cost and by what a measurement can tell apart.

## Rule 1: native install without Docker Desktop

2 means a native install on macOS. 1 means containers only, which can run on Colima or Podman as the map allows. 0 means Docker Desktop is required.

| Candidate | Score | Evidence |
|---|---|---|
| Keiko | 2 | Desktop downloads from the releases page and `npm install -g @oscharko-dev/keiko` (`Keiko:README.md`, "Download"). The lab script runs it natively next to `mlx-lm` and LiteLLM (`scripts/testing/review-quality/lab/README.md` at `b5fe2a8`). |
| Open WebUI | 2 | The docs offer Python installs with pip, uv, venv and Conda next to Docker ([quick start](https://github.com/open-webui/docs/blob/main/docs/getting-started/quick-start/index.mdx)). Python must be 3.11 or 3.12 ([pyproject.toml](https://github.com/open-webui/open-webui/blob/main/pyproject.toml)). |
| LightRAG | 2 | `uv tool install "lightrag-hku[api]"` or pip ([README](https://github.com/HKUDS/LightRAG/blob/main/README.md)). |
| SurfSense | 2 | A desktop app for macOS Apple Silicon, Windows and Linux ([README](https://github.com/MODSetter/SurfSense)). |
| PrivateGPT | 2 | `brew install private-gpt` ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). |
| AnythingLLM | 1 | The bare-metal install exists, but it "is **not supported** by the core-team" ([BARE_METAL.md](https://github.com/Mintplex-Labs/anything-llm/blob/master/BARE_METAL.md)). The native Desktop app is proprietary, so glue code cannot change it. |
| Dify CE | 1 | `docker compose up -d` ([README](https://github.com/langgenius/dify/blob/main/README.md)). The source-code setup is for contributors. |
| Onyx CE | 1 | Only Docker Compose, Helm, Terraform and AWS ECS appear in [`deployment/`](https://github.com/onyx-dot-app/onyx/tree/main/deployment). The default compose file starts 13 services, including Postgres, OpenSearch, Redis, two model servers and MinIO ([docker-compose.yml](https://github.com/onyx-dot-app/onyx/blob/main/deployment/docker_compose/docker-compose.yml)). |
| RAGFlow | 1 | Docker images exist for Linux x86-64 and ARM64. "macOS is not currently supported by the Go backend" ([build docs](https://github.com/infiniflow/ragflow/blob/main/docs/develop/build_docker_image.mdx)). Launching from source still starts Elasticsearch, MySQL, MinIO, NATS and Kvrocks with Docker Compose ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). |

## Rule 2: fits next to the models on a 64 GB Mac

Memory left for the app:

- The lab pair needs 38.8 GB of weights at 4 bit, and the two model servers keep bounded prompt caches of 4 GB and 6 GB (`scripts/testing/review-quality/lab/lab.sh` at `b5fe2a8`, lines 33-34 and 215-216). That is 48.8 GB.
- A small embedding model and reranker add about 2.4 to 3.5 GB. Qwen3-Embedding-0.6B and Qwen3-Reranker-0.6B have 1.19 GB of weights each, and bge-reranker-v2-m3 has 2.27 GB (Hugging Face model API, 2026-11-17).
- That leaves about 12 GB for macOS, LiteLLM, the browser and the app. If macOS and the browser take about 4 GB (my assumption, not measured), the app gets about 8 GB. Containers on Colima also pay for the VM.

Side finding for ticket #4: macOS lets the GPU use about three quarters of RAM by default (`scripts/testing/review-quality/lab/README.md` at `b5fe2a8`), which is about 48 GB on a 64 GB Mac. The 48.8 GB of weights plus full prompt caches is just above that.

2 means one process or a small set with no documented need above 4 GB. 1 means a documented minimum that fits only with little headroom. 0 means a documented minimum above the 8 GB budget.

| Candidate | Score | Evidence |
|---|---|---|
| Keiko | 2 | One Node.js server with SQLite (`node:sqlite`). Its vector cache is capped at 256 MiB per index and in total, against a 512 MiB RSS-delta envelope (`Keiko:docs/adr/README.md`, ADR-0164 row). Open risk: issue #4456 measured 4.27 GB and a crash for one connected-folder question over 10,600 files. |
| Open WebUI | 2 | One Python process with SQLite and an embedded vector store by default. No vendor minimum found. With an external embedding engine it loads no local model (`RAG_EMBEDDING_ENGINE` in [config.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/config.py)). Not measured. |
| LightRAG | 2 | One Python process. Default storages are local files under `WORKING_DIR` ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). Not measured. |
| AnythingLLM | 2 | "aim for at least 2GB of RAM" ([BARE_METAL.md](https://github.com/Mintplex-Labs/anything-llm/blob/master/BARE_METAL.md)). |
| SurfSense | 2 | One desktop app with a local index. It runs its embedding model in-process (`surfsense_local/backend/modules/embedding`), which costs memory next to the lab models. Not measured. |
| PrivateGPT | 2 | One API process that "does **not** run models itself" ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). Not measured. |
| Dify CE | 1 | "RAM >= 4 GiB" ([README](https://github.com/langgenius/dify/blob/main/README.md)), spread over about ten containers plus the Colima VM. |
| Onyx CE | 0 | Onyx Standard needs at least 10 GB RAM, preferred 16 GB or more. The per-container figures add up to 24 GiB, with 8 GiB for the background workers alone ([resourcing](https://docs.onyx.app/deployment/getting_started/resourcing)). Onyx Lite needs 2 to 4 GB but turns off "connectors and RAG search" ([docker-compose.onyx-lite.yml](https://github.com/onyx-dot-app/onyx/blob/main/deployment/docker_compose/docker-compose.onyx-lite.yml)), so it cannot be a project memory. |
| RAGFlow | 0 | "Recommended starting configuration: 4 CPU cores, 16 GB RAM" ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). |

## Rule 3: talks to one OpenAI-compatible gateway

The app must send chat, embedding and rerank calls to LiteLLM. 2 means all three. 1 means one of them cannot go through the gateway.

| Candidate | Score | Evidence |
|---|---|---|
| Keiko | 2 | OpenAI-compatible `/chat/completions` and `/embeddings` adapters, and a "LiteLLM/Cohere-compatible rerank adapter" (`Keiko:packages/keiko-model-gateway/src/openai-adapter.ts`, `openai-embedding-adapter.ts`, `rerank-adapter.ts`). The review lab already runs Keiko against LiteLLM 1.106.6 with the lab pair. |
| Open WebUI | 2 | OpenAI connections for chat. `RAG_EMBEDDING_ENGINE` with `RAG_OPENAI_API_BASE_URL` for embeddings, and `RAG_EXTERNAL_RERANKER_URL` for reranking ([config.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/config.py)). |
| LightRAG | 2 | OpenAI bindings for LLM and embeddings. `RERANK_BINDING=cohere` with a custom `RERANK_BINDING_HOST` for reranking ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). |
| Dify CE | 2 | The OpenAI-API-compatible model plugin has `llm`, `text_embedding` and `rerank` model types ([plugin source](https://github.com/langgenius/dify-official-plugins/tree/main/models/openai_api_compatible/models)). |
| Onyx CE | 2 | LLM calls go through the litellm library. Embedding providers include `litellm` and `openai_compatible`, and reranker providers include `litellm` ([shared_configs/enums.py](https://github.com/onyx-dot-app/onyx/blob/main/backend/shared_configs/enums.py)). |
| RAGFlow | 2 | The "OpenAI-API-Compatible" provider maps chat, embedding and rerank to `chat/completions`, `embeddings` and `rerank` ([openai-api-compatible.json](https://github.com/infiniflow/ragflow/blob/main/conf/models/openai-api-compatible.json)). |
| AnythingLLM | 1 | LiteLLM providers exist for chat and embeddings ([AiProviders](https://github.com/Mintplex-Labs/anything-llm/tree/master/server/utils/AiProviders), [EmbeddingEngines](https://github.com/Mintplex-Labs/anything-llm/tree/master/server/utils/EmbeddingEngines)). The only reranker is `native`, which runs in the app process ([EmbeddingRerankers](https://github.com/Mintplex-Labs/anything-llm/tree/master/server/utils/EmbeddingRerankers)). |
| SurfSense | 1 | Chat takes "Any OpenAI-compatible API" ([README](https://github.com/MODSetter/SurfSense)). Embeddings run in-process from Hugging Face models (`surfsense_local/backend/modules/embedding/huggingface`). |
| PrivateGPT | 1 | Connects to any server with `/v1/chat/completions` and `/v1/models` through `OPENAI_API_BASE`, and lists embeddings as supported ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). The README does not mention reranking. |

## Rule 4: ingests GitHub repo files, issues and pull requests

2 means all three from GitHub. 1 means some of them. 0 means no GitHub source.

| Candidate | Score | Evidence |
|---|---|---|
| Dify CE | 2 | The GitHub datasource plugin reads files through the contents API and fetches issues and pull requests with their comments ([github.py](https://github.com/langgenius/dify-official-plugins/blob/main/datasources/github/datasources/github.py), lines 276, 394-419 and 585-626). |
| Onyx CE | 1 | The GitHub connector has `include_prs`, `include_issues` and `include_files` flags. Files are limited to prose: `.md`, `.mdx`, `.rst`, `.txt` and extensionless names such as README (`GITHUB_INDEXABLE_FILE_EXTENSIONS` in [connector.py](https://github.com/onyx-dot-app/onyx/blob/main/backend/onyx/connectors/github/connector.py)). Source code is not indexed. Ticket #5 found that comments and review comments are not indexed either. |
| Keiko | 1 | A repository Knowledge Pod indexes a local working tree (`Keiko:packages/keiko-local-knowledge/src/repository-pod.ts`). Source scopes are only `folder`, `repository` and `files` (`Keiko:packages/keiko-contracts/src/local-knowledge.ts`). Issues and pull requests are read live through `gh api` for coding context, not indexed (`Keiko:packages/keiko-server/src/coding-context/githubCodeContextPort.ts`). |
| Open WebUI | 1 | The oikb GitHub connector reads the git tree and file contents only ([github.py](https://github.com/open-webui/oikb/blob/main/src/oikb/connectors/github.py)). |
| AnythingLLM | 1 | The GitHub loader reads a branch's files with ignore paths. It has no issue or pull request code ([RepoLoader](https://github.com/Mintplex-Labs/anything-llm/blob/master/collector/utils/extensions/RepoLoader/GithubRepo/RepoLoader/index.js)). |
| RAGFlow | 1 | The GitHub connector "reads GitHub issues and pull requests", not files, and stores each item's `body` without a comments request ([github.go](https://github.com/infiniflow/ragflow/blob/main/internal/syncer/connector/github.go)). Only the GitLab connector reads code files (background report). |
| LightRAG | 0 | Upload and input folder only (`INPUT_DIR` in [env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). |
| SurfSense | 0 | Files and folders. The GitHub connector docs sit under `docs/legacy` ([tree](https://github.com/MODSetter/SurfSense/tree/main/www/content/docs/legacy/connectors/external)). |
| PrivateGPT | 0 | File ingestion and MCP connectors, no GitHub connector ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). |

No own connector covers code, issues, pull requests and comments together. Ticket #5 therefore plans a separate ingestion job that chunks code with tree-sitter and syncs issues and pull requests through the REST API. Rule 4b scores how well each app takes that job's output.

## Rule 4b: accepts externally produced chunks with citation URLs

2 means the app reads chunks produced outside it and shows each chunk's own URL as the citation, with no app changes. 1 means the app takes external text through an API or a folder, but re-chunks it, or shows a file name instead of the URL, or needs code for the URL. 0 means no ingestion API found.

| Candidate | Score | Evidence |
|---|---|---|
| Open WebUI | 2 | An external knowledge base reads a Qdrant, Milvus or pgvector collection (`EXTERNAL_KNOWLEDGE_PROVIDERS` in [knowledge.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/routers/knowledge.py), line 576). Configurable `content_field`, `source_field` and `url_field` become the citation's name, source and URL ([retrieval/external.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/retrieval/external.py), lines 45-62). The job must embed chunks with the same model that Open WebUI uses for queries. Not verified: whether hybrid search and the reranker also apply to external knowledge bases. |
| Onyx CE | 2 | `POST /onyx-api/ingestion` upserts a document whose sections each carry a `link` ([ingestion.py](https://github.com/onyx-dot-app/onyx/blob/main/backend/onyx/server/onyx_api/ingestion.py), [connectors/models.py](https://github.com/onyx-dot-app/onyx/blob/main/backend/onyx/connectors/models.py)). |
| Keiko | 1 | Connector pods mount fetched items as files and index them with the normal chunker. Each item's stored `webui_path`, joined to the connector's base URL, becomes the citation target (`Keiko:packages/keiko-local-knowledge/src/connector-pod.ts`, `Keiko:packages/keiko-local-knowledge/src/connector-source-metadata.ts`). This exists only for Atlassian providers, and Keiko re-chunks everything. A GitHub provider would reuse the same path. |
| LightRAG | 1 | `POST /documents/text` takes a `file_source`, which LightRAG keeps "for citation" (`file_paths` in [lightrag.py](https://github.com/HKUDS/LightRAG/blob/main/lightrag/lightrag.py)). LightRAG re-chunks the text. Inserting pre-made chunks goes through `ainsert_custom_chunks`, marked deprecated and without a source. Not verified: whether a full URL survives `normalize_file_path`. |
| Dify CE | 1 | The service API creates documents from text and adds segments (chunks) to a document (`/datasets/<id>/documents/<id>/segments` in [segment.py](https://github.com/langgenius/dify/blob/main/api/controllers/service_api/dataset/segment.py)). Citations show the document name. A URL per segment was not verified. |
| RAGFlow | 1 | The HTTP API has "Add chunk" ([http_api_reference.md](https://github.com/infiniflow/ragflow/blob/main/docs/references/http_api_reference.md)). Citations show the document. |
| AnythingLLM | 1 | The developer API uploads raw text with `title`, `docAuthor`, `description` and `docSource` metadata ([document API](https://github.com/Mintplex-Labs/anything-llm/blob/master/server/endpoints/api/document/index.js)). The app chunks it again. |
| PrivateGPT | 1 | "File and artifact ingestion" through the API ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). |
| SurfSense | 0 | Files and folders only. No ingestion API found. |

Glue that works for every candidate with folder or text ingestion: write one Markdown file per issue, pull request or code unit with its GitHub URL in front matter. The app then re-chunks it, and the citation shows the file unless the app reads the URL.

## Rule 5: answers with citations

All scored candidates get 2. The difference that matters is whether the app checks that a citation supports its claim, and only Keiko does.

| Candidate | Score | Evidence |
|---|---|---|
| Keiko | 2 | Inline `[n]` markers map to retrieval references (`Keiko:packages/keiko-local-knowledge/src/conversation/citation-attacher.ts`). An optional entailment stage asks a judge model whether each claim is supported (`Keiko:packages/keiko-server/src/grounded-entailment-stage.ts`). This stage is a natural place for the checker model of the lab pair. |
| Open WebUI | 2 | Retrieved sources appear as citations (background report, comparison table). |
| LightRAG | 2 | "LightRAG now supports citation functionality" since 2025-03 ([README](https://github.com/HKUDS/LightRAG/blob/main/README.md)). |
| Dify CE | 2 | Citations and attributions (background report, comparison table). |
| AnythingLLM | 2 | "Intuitive chat UI with drag-and-drop uploads and source citations" ([README](https://github.com/Mintplex-Labs/anything-llm/blob/master/README.md)). |
| SurfSense | 2 | "Research (cited answers)" for every listed format ([README](https://github.com/MODSetter/SurfSense)). |
| Onyx CE | 2 | Citations (background report, comparison table). |
| RAGFlow | 2 | Grounded citations ([README](https://github.com/infiniflow/ragflow/blob/main/README.md)). |
| PrivateGPT | 2 | "Retrieval with citations" ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). |

## Rule 6: exports and imports one project memory

2 means a built-in export and import of one project memory with its index. 1 means a partial export, or all state in one directory that a copy can move. 0 means no export.

| Candidate | Score | Evidence |
|---|---|---|
| Open WebUI | 1 | `GET /knowledge/{id}/export` is admin-only and returns "a zip file containing .txt files" of extracted text ([knowledge.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/routers/knowledge.py)). Vectors, metadata and source URLs are lost, and there is no matching import route. |
| LightRAG | 1 | No export feature. With the default file storages, all state of one instance lives in `WORKING_DIR` ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)), so copying that folder moves it. The embedding model must stay the same (background report). |
| Keiko | 0 | A QA record states "There is no mounted capsule-export capability for either kind" (`Keiko:docs/qa/jira-chat-pods-4081.md`, AC7). All pods share one encrypted `capsules.db` per state directory, under the fixed namespace `knowledge` (`Keiko:packages/keiko-local-knowledge/src/store-paths.ts`, `Keiko:packages/keiko-server/src/local-knowledge-store-open.ts`). The key comes from an env var, the macOS Keychain or a keyfile (`Keiko:packages/keiko-server/src/localKnowledgeKeyProvider.ts`), so an export must also re-key or carry a key. No open issue plans an export; epic #2558 (shared pod curation) is the closest. |
| AnythingLLM | 0 | The only export in the frontend is chat export. The feature request to export and import the document library is open ([#4660](https://github.com/Mintplex-Labs/anything-llm/issues/4660)). |
| SurfSense | 0 | No workspace export found. The only import module is for chat sessions (`surfsense_local/backend/modules/agent/agent_threads/session_import.py`). |
| Dify CE | 0 | No knowledge base data export found. Pipelines and apps export as DSL configuration only. |
| Onyx CE | 0 | No document or index export in `backend/onyx`. The `ee` folder has only log and usage exports ([tree](https://github.com/onyx-dot-app/onyx/tree/main/backend/ee/onyx/server)). |
| RAGFlow | 0 | Export code exists for agents, MCP settings and chat messages, not datasets ([import and export guide](https://github.com/infiniflow/ragflow/blob/main/docs/guides/agent/import_and_export_workflows.md)). |
| PrivateGPT | 0 | The README lists no export. |

No candidate has what the map asks for. Ticket #6 defines the bundle. For single-process apps with one state folder (Keiko, Open WebUI, LightRAG, AnythingLLM), running one app instance per project memory makes export a folder copy plus a manifest. Multi-database stacks (Onyx, RAGFlow, Dify) would need coordinated dumps of Postgres or MySQL, the search engine and the object store.

## Rule 7: enforces a model policy per project memory

The glossary defines the model policy as "the furthest model location one project memory may send its content to", with on-device only as the default. 2 means the app enforces that for chat, embeddings and reranking. 1 means the app enforces some per-project-memory model rule that glue can extend. 0 means model choice is per instance or per chat, and nothing ties it to the project memory.

No candidate knows model locations. Every one needs a location label per LiteLLM route.

| Candidate | Score | Evidence |
|---|---|---|
| Keiko | 1 | Each Knowledge Pod carries a model-use policy with modes `sealed-local`, `standard` and `custom` and seven operations, such as `externalEmbeddings` and `answerSynthesis` (`Keiko:packages/keiko-contracts/src/local-knowledge-model-use-policy.ts`). The code checks it at indexing, retrieval and answering, and fails closed (`Keiko:packages/keiko-local-knowledge/src/indexing/orchestrator.ts`, `Keiko:packages/keiko-server/src/local-knowledge-grounded-qa.ts`). The policy is keyed by operation, not by model location. Every gateway embedding counts as external (`locality: "provider"` in `Keiko:packages/keiko-local-knowledge/src/knowledge-pods.ts`), and `localEmbeddings` has no call site. So `sealed-local` blocks indexing with embeddings and blocks all answers, even when the models run on-device. In flight: issue #4338 adds an explicit, fail-closed embedding model choice, which still knows no locations. |
| Dify CE | 1 | Each app pins its model, and each knowledge base has its own embedding model (background report). Nothing knows locations. |
| AnythingLLM | 1 | Each workspace stores its own `chatProvider` and `chatModel` ([schema.prisma](https://github.com/Mintplex-Labs/anything-llm/blob/master/server/prisma/schema.prisma)). The embedder is global. |
| RAGFlow | 1 | Each dataset chooses its embedding model and each chat assistant its LLM (background report). |
| Open WebUI | 0 | Embedding and reranking settings are global (`RAG_EMBEDDING_ENGINE`, `RAG_RERANKING_ENGINE` in [config.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/config.py)). A custom model can bundle a base model and knowledge, but nothing stops a chat from attaching a knowledge base to another model. |
| SurfSense | 0 | Outbound hosts need consent, but the consent is per host for the whole app, not per workspace (`surfsense_local/backend/modules/egress/service.py`). |
| Onyx CE | 0 | The flag that restricts an LLM provider to agents or groups is marked "EE only" (`is_public` on `LLMProvider` in [models.py](https://github.com/onyx-dot-app/onyx/blob/main/backend/onyx/db/models.py)). The embedding model is instance-wide. |
| LightRAG | 0 | One LLM, embedding and rerank binding per instance ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). |
| PrivateGPT | 0 | One `OPENAI_API_BASE` per instance ([README](https://github.com/zylon-ai/private-gpt/blob/main/README.md)). |

Two glue options apply to every candidate:

- Enforce in LiteLLM. Give each project memory its own LiteLLM key, limited to the routes its model policy allows. LiteLLM virtual keys take a `models` list, but they need a Postgres database ([virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys)). This works only if the app can use a different key per project memory, which in practice means one app instance per project memory, or Keiko-style per-pod checks in the app. Ticket #3 owns the LiteLLM setup.
- Enforce in the app. Only Keiko has the per-pod check points already. The change is to replace the operation flags with a location ceiling and to look up each route's location.

## Shortlist for the head-to-head trial

The trial in ticket #12 should run these three against the same corpus, the same gold questions from ticket #11, the same LiteLLM routes and the same embedding model and reranker. The three differ in retrieval design, so the measurement can tell them apart.

### Keiko from `dev`

Why measure it:

- It has hybrid retrieval with lexical and vector lanes and reranking, plus an entailment check that can use the checker model.
- It is the only candidate with the hardest rule half done: the per-pod policy is checked at indexing, retrieval and answering, and it fails closed.
- It already runs against LiteLLM with the exact lab pair, natively on a Mac.

Conditions for a fair run:

- Use a build that includes the #4456 and #4458 fixes, or record that they are missing. Without #4456 a large repo may crash the server. Without #4458 multi-question sessions can answer from the wrong file.
- State whether the trial uses Knowledge Pods or connected-folder search. Epic #4429 changes only the second.
- Use the `standard` policy mode for the trial. `sealed-local` refuses answers even with on-device models.

Glue it needs after a win:

- Re-key the model policy from operations to a model location ceiling, and label LiteLLM routes with a location.
- Build pod export and import, including key handling for the encrypted store.
- Take the ticket #5 ingestion output through a GitHub provider on the existing connector-pod path, so citations point to GitHub URLs.

Open risks:

- One person wrote 1,059 of the 1,146 commits on `dev` (`git log` on 2026-11-17).
- The product covers far more than a project memory: a coding agent, Jira and Confluence write tools, and voice. The reference installation would carry that code or need a cut-down build.
- The trademark policy bars shipping a fork under the Keiko name.
- Export touches the content encryption (ADR-0047), so it is a design task, not a file copy.

### Open WebUI with hybrid search

Why measure it:

- It is the most used open-source chat front end in this list (158,932 GitHub stars on 2026-11-17). It is the baseline any custom stack has to beat.
- `ENABLE_RAG_HYBRID_SEARCH` turns on BM25 plus vectors, and the external reranker goes through LiteLLM ([config.py](https://github.com/open-webui/open-webui/blob/main/backend/open_webui/config.py)).
- It installs with pip and keeps state in one folder.
- Its external knowledge bases read chunks and URLs straight from a vector collection, so the ticket #5 ingestion job can feed it without app changes.

Conditions for a fair run:

- Check whether hybrid search and the reranker apply to external knowledge bases. If they do not, run it once with uploaded Markdown files (hybrid on) and once with the external collection, and record both.

Glue it needs after a win:

- A per-project-memory model policy. Either run one instance per project memory with its own LiteLLM key, or add a filter that blocks other models.
- Export and import with vectors and metadata. With external knowledge bases, the export is the collection plus a manifest. The built-in zip holds plain text only.
- A vector store next to it (Qdrant as a native binary, or pgvector in a native Postgres) if the external path is used.

Open risks:

- The branding clause and the contributor license agreement let the vendor change terms later.
- Chunking is character or token based, with no code awareness (background report).
- One instance per project memory means several Python processes if a consultant opens two at once.

### LightRAG

Why measure it:

- It is the graph-based design in the trial. It builds an entity and relation graph next to vector search, which may help "which decision replaced which" questions.
- It is the only graph engine in the survey with a server, a web UI, citations, a 2026 release and a native install.
- It works with LiteLLM for all three model types.

Glue it needs after a win:

- All GitHub ingestion.
- One instance per project memory for both export and model policy.
- A consultant-facing UI, because its web UI is aimed at inspecting the graph.

Open risks:

- Graph extraction calls the LLM for every chunk at indexing time. With a 35B-A3B model on one Mac, indexing a large repo may take hours. This is unmeasured, so the trial should record indexing time.
- The example config binds to `0.0.0.0` and runs without auth unless `LIGHTRAG_API_KEY` or `AUTH_ACCOUNTS` is set ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)).
- Its published results are LLM-judged win rates, not accuracy, so they predict little.

### Left out of the trial

- Dify CE. It has the best GitHub ingestion among the apps that fit, but it runs about ten containers in an 8 GB budget and has no data export. Measure it only if the three above all fail on GitHub-heavy questions.
- Onyx CE and RAGFlow. They do not fit next to the models on one 64 GB Mac. Onyx's ingestion API would otherwise suit the ticket #5 job.
- AnythingLLM. Its native app is proprietary, its MIT bare-metal install is unsupported, and its reranker cannot go through LiteLLM.
- SurfSense. Its embeddings cannot go through LiteLLM and it has no GitHub source.
- PrivateGPT. It is an API without an end-user UI or connectors.
- HippoRAG 2 and FlashRAG. They have the best published multi-hop scores, but they are research code without citations or a UI, and HippoRAG has no 2026 release. If the trial needs a retrieval ceiling, FlashRAG's naive and reranked pipelines are the cheapest reference to add.

## Sources

Keiko (`origin/dev` at `d72d6db6f`, 2026-11-17, unless marked):

- `LICENSE`, `TRADEMARKS.md`, `README.md`, `package.json`
- `packages/keiko-contracts/src/local-knowledge-model-use-policy.ts`, `packages/keiko-contracts/src/local-knowledge.ts`
- `packages/keiko-local-knowledge/src/model-use-policy.ts`, `knowledge-pods.ts`, `repository-pod.ts`, `store-paths.ts`, `indexing/orchestrator.ts`, `retrieval/scoped-vector-search.ts`, `conversation/citation-attacher.ts`, `parsers/code-parser.ts`
- `packages/keiko-server/src/local-knowledge-grounded-qa.ts`, `grounded-entailment-stage.ts`, `local-knowledge-store-open.ts`, `localKnowledgeKeyProvider.ts`, `coding-context/githubCodeContextPort.ts`
- `packages/keiko-model-gateway/src/openai-adapter.ts`, `openai-embedding-adapter.ts`, `rerank-adapter.ts`
- `scripts/check-retrieval-quality.mjs`, `scripts/check-grounded-retrieval-quality.mjs`
- `docs/adr/README.md` (ADR-0164), `docs/qa/jira-chat-pods-4081.md`, `docs/local-knowledge/knowledge-pods.md`
- `scripts/testing/review-quality/lab/README.md` and `lab.sh` at `b5fe2a8` (branch `claude/4130-counter-check-retry`)
- Issues: https://github.com/oscharko-dev/Keiko/issues/4429, 4430, 4431, 4434, 4338, 4456, 4457, 4458, 2558, 2559, 2560

Other candidates (GitHub API and docs, 2026-11-17):

- Open WebUI: https://github.com/open-webui/open-webui (LICENSE, pyproject.toml, backend/open_webui/config.py, backend/open_webui/routers/knowledge.py), https://github.com/open-webui/docs, https://github.com/open-webui/oikb
- LightRAG: https://github.com/HKUDS/LightRAG (README.md, env.example), https://arxiv.org/abs/2410.05779
- Dify: https://github.com/langgenius/dify (README.md, LICENSE), https://github.com/langgenius/dify-official-plugins (datasources/github, models/openai_api_compatible)
- AnythingLLM: https://github.com/Mintplex-Labs/anything-llm (README.md, BARE_METAL.md, server/prisma/schema.prisma, server/utils, collector RepoLoader, issue #4660), https://docs.anythingllm.com/installation-desktop/overview, https://docs.anythingllm.com/installation-desktop/terms
- SurfSense: https://github.com/MODSetter/SurfSense (README.md, surfsense_local/backend/modules)
- Onyx: https://github.com/onyx-dot-app/onyx (deployment/docker_compose, backend/onyx/connectors/github/connector.py, backend/shared_configs/enums.py, backend/onyx/db/models.py), https://docs.onyx.app/deployment/getting_started/resourcing
- RAGFlow: https://github.com/infiniflow/ragflow (README.md, docs/develop/build_docker_image.mdx, internal/syncer/connector/github.go, conf/models/openai-api-compatible.json)
- PrivateGPT: https://github.com/zylon-ai/private-gpt (README.md)
- Ticket #5 findings: https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md (Onyx and RAGFlow connector scope re-checked in source here; Sourcebot license status taken from it, with the missing LICENSE file checked here)
- Survey repositories: every repository linked in the survey tables, read through the GitHub API (license, last push, latest release, README)
- Benchmark papers: HippoRAG 2 https://arxiv.org/abs/2502.14802, LightRAG https://arxiv.org/abs/2410.05779, FlashRAG README result table
- LiteLLM: https://docs.litellm.ai/docs/proxy/virtual_keys
- Model sizes: Hugging Face model API for Qwen/Qwen3-Embedding-0.6B, Qwen/Qwen3-Reranker-0.6B, BAAI/bge-reranker-v2-m3
- Background report: `docs/research/rag-stack-landscape.md` on branch `research/rag-stack-landscape`
