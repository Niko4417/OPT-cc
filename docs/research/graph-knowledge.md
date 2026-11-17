# Where graph-based knowledge helps the project memory

Research for [issue #15](https://github.com/Niko4417/OPT-cc/issues/15), part of the map in [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). Sources checked on 2026-11-17. Repository versions and licenses come from the GitHub REST API on that date.

## Question

Where would a knowledge graph improve answers over plain hybrid retrieval in a project memory? The candidate question types are tracing a decision record to the code and issues it affects, following which decision supersedes which, answering questions across many documents, and mapping code dependencies. What do papers, independent benchmarks and maintained tools show, what does indexing cost on a 64 GB Mac with the local lab models, and should a graph candidate join the head-to-head measurement in [issue #13](https://github.com/Niko4417/OPT-cc/issues/13)?

## Answer

Use a graph for specific question types, and build it without a language model.

- Do not make an LLM-extracted entity graph (GraphRAG, LightRAG, HippoRAG, KAG, Graphiti) the core of the project memory. Three independent studies find that these graphs lose to plain retrieval on single-fact questions and win only on multi-hop and some summary questions ([Han et al.](https://arxiv.org/abs/2502.11371), [GraphRAG-Bench, Xiang et al.](https://arxiv.org/abs/2506.05690), [HippoRAG 2 paper](https://arxiv.org/abs/2502.14802)). The large win rates that GraphRAG and LightRAG report come from pairwise LLM judging, and a debiased re-run turned LightRAG's 72% win over naive RAG into a slight loss ([Zeng et al.](https://arxiv.org/abs/2506.06331)).
- Indexing all of `thunderbird/thunderbird-android` (about 5.2 million tokens of code, docs and issue and pull request bodies) with LightRAG would take about 25 hours of local generation on Qwen3.6-35B-A3B. HippoRAG 2 would take about 17 hours and Microsoft GraphRAG about 40 hours. These are estimates from published token ratios, not measurements (section "Indexing cost").
- Two graphs pay off and cost no LLM calls. The first is a link graph that the ingestion job builds from decision record metadata and explicit references: `status`, `supersedes`, dates, issue and pull request numbers, and the files a pull request changed. It answers "which decision is current" and "which code and issues does this decision affect". The second is a code dependency graph from static analysis (tree-sitter, SCIP or a code property graph). It answers "what calls or depends on X". Both rebuild in seconds to minutes and update incrementally.
- Keep LightRAG in the head-to-head as the LLM-graph contrast, but index only Markdown docs, issues and pull requests for it, and score every stack per question type. Add HippoRAG 2 as a retrieval-only arm, because it is the only LLM-graph method that matches plain retrieval on simple questions in independent tests. Do not add Microsoft GraphRAG, KAG or Graphiti (section "Recommendation").

## Evidence by question type

The table separates the authors' claims about their own method from results by other groups. "Plain RAG" means dense or hybrid chunk retrieval.

| Question type | Project memory example | Author claims | Independent results | Verdict |
|---|---|---|---|---|
| Single fact | "Which database library did RFC 0007 choose?" | HippoRAG 2 claims parity or better than plain RAG ([paper](https://arxiv.org/abs/2502.14802)) | Plain RAG wins or ties against every graph method in three studies (below) | No graph |
| Multi-hop across documents | "Which pull requests implement the decision that replaced the per-account stores?" | HippoRAG: up to 20% better on multi-hop ([paper](https://arxiv.org/abs/2405.14831)). KAG: 19.6% and 33.5% relative F1 gains ([paper](https://arxiv.org/abs/2409.13731)) | HippoRAG 2 and GraphRAG local search beat plain RAG by 1 to 3 points on HotpotQA and MultiHop-RAG ([Han et al.](https://arxiv.org/abs/2502.11371)). HippoRAG 2 leads complex reasoning on GraphRAG-Bench, 53.4 against 42.9 ([Xiang et al.](https://arxiv.org/abs/2506.05690)) | Small gain from LLM graphs; explicit links cover the project memory's main case |
| Global summary | "What risks came up across all workshops?" | GraphRAG: 72 to 83% comprehensiveness win rate ([paper](https://arxiv.org/abs/2404.16130)). LightRAG: 61 to 85% overall win rate ([paper](https://arxiv.org/abs/2410.05779)) | After order averaging, judges prefer plain RAG for comprehensiveness and GraphRAG global for diversity ([Han et al.](https://arxiv.org/abs/2502.11371)). Debiasing erases LightRAG's win ([Zeng et al.](https://arxiv.org/abs/2506.06331)). GraphRAG local search leads contextual summarization on GraphRAG-Bench, 64.4 against 51.3 ([Xiang et al.](https://arxiv.org/abs/2506.05690)) | Mixed; a query-time map-reduce over the filtered documents is cheaper |
| Temporal, "which decision is current" | "Is ADR 0010 still the plan, or was it superseded?" | Zep: temporal reasoning 54.1% against 36.5% for full context with gpt-4o-mini ([paper](https://arxiv.org/abs/2501.13956)) | GraphRAG community search beats plain RAG on MultiHop-RAG temporal questions, 53.3 against 30.7 ([Han et al.](https://arxiv.org/abs/2502.11371)). Zep scores below Mem0 on LoCoMo temporal questions, 49.3 against 55.5, in a competitor's paper ([Mem0](https://arxiv.org/abs/2504.19413)) | Metadata link graph; decision records state their status |
| Code dependency | "What breaks if `legacy/storage` is removed?" | RepoGraph: SWE-bench Lite gains of 2.0 to 2.7 points ([paper](https://arxiv.org/abs/2410.14684)). LocAgent: 92.7% file-level localization ([paper](https://arxiv.org/abs/2503.09089)) | StackRepoQA: graph retrieval scores 6.12 against 5.66 for file retrieval on 134 Java projects (1 to 10 scale) ([Alebachew et al.](https://arxiv.org/abs/2603.26567)) | Static-analysis code graph |

### Single fact questions

Plain retrieval wins this type in every independent comparison found:

- Han et al. ran one protocol for all methods with Llama-3.1-8B as the reader. On NQ, plain RAG scored 64.78 F1, GraphRAG local 63.01, HippoRAG 2 61.03, RAPTOR 60.04 and a triplet graph 34.28 ([arXiv 2502.11371](https://arxiv.org/abs/2502.11371)). Their summary: "RAG is consistently effective for single-hop, detail-oriented queries that require precise evidence".
- GraphRAG-Bench (Novel set, GPT-4o-mini) scored fact retrieval at 60.92 for plain RAG with reranking, 60.14 for HippoRAG 2, 58.62 for LightRAG and 49.29 for Microsoft GraphRAG ([arXiv 2506.05690](https://arxiv.org/abs/2506.05690), v3 of 2026-02-22). The authors attribute the losses to graph processing adding "redundant or noisy information".
- The HippoRAG 2 paper, which reran all baselines with Llama-3.3-70B, reports NQ F1 of 61.9 for plain retrieval with NV-Embed-v2, 63.3 for HippoRAG 2, 46.9 for GraphRAG and 50.7 for RAPTOR ([arXiv 2502.14802](https://arxiv.org/abs/2502.14802), Table 2). Its LightRAG row (16.6 on NQ, 1.6 on MuSiQue) is far below every other study and probably reflects a configuration problem, so it is left out of the verdict.

On "unanswerable" questions, graph context also hurts. On MultiHop-RAG null queries, plain RAG refused correctly 96.01% of the time against 80.07% for GraphRAG local and 19.27% for GraphRAG global ([Han et al.](https://arxiv.org/abs/2502.11371)). Refusal is a primary metric in the evaluation setup from ticket #7.

### Multi-hop questions

Graph methods gain a little, and only some of them:

- Han et al.: HotpotQA F1 63.01 for HippoRAG 2, 61.66 for GraphRAG local and 60.04 for plain RAG. MultiHop-RAG accuracy 70.27, 69.01 and 67.02. 13.6% of MultiHop-RAG queries were answered only by GraphRAG and 11.6% only by plain RAG, so the two are complementary. Routing or merging both raised the best baseline by 1.1% and 6.4% with Llama-3.1-70B ([arXiv 2502.11371](https://arxiv.org/abs/2502.11371)).
- GraphRAG-Bench: complex reasoning 53.38 for HippoRAG 2, 50.93 for Microsoft GraphRAG, 49.07 for LightRAG and 42.93 for plain RAG. In a scaling test, plain RAG fell from 58.64% to 43.20% as the corpus grew from about 56k to 1,132k tokens, while HippoRAG 2 stayed at about 54% ([arXiv 2506.05690](https://arxiv.org/abs/2506.05690)).
- Graph quality depends on the extraction model. Han et al. measured MultiHop-RAG accuracy of 65.77 for plain RAG, 71.17 with a graph built by GPT-4o-mini and 75.08 with GPT-4o. GraphRAG-Bench found that HippoRAG 2 gained 6.8 points from Qwen2.5-3B to 14B, against 3.0 points for plain RAG, and names 7B as a practical minimum. No study used a 35B-A3B mixture-of-experts model.

For a project memory, the frequent multi-hop chains run through explicit references: a decision record names an issue, the issue links a pull request, and the pull request changes files. Thunderbird's records already do this. ADR 0001 links PR #7221, ADR 0009 links issue #10197 and PR #10198, and RFC 0007 links issues #11104 and #11195 ([test dataset research](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md)). A parser follows those links exactly. An LLM extractor would have to rediscover them from prose.

### Global summary questions

GraphRAG was built for "global sensemaking" questions over a whole corpus. Its paper reports comprehensiveness win rates of 72 to 83% on podcast transcripts (about 1 million tokens) and 72 to 80% on news (about 1.7 million tokens) against vector RAG, judged pairwise by gpt-4-turbo. Vector RAG gave the most direct answers in every comparison ([arXiv 2404.16130](https://arxiv.org/abs/2404.16130)). LightRAG reports overall win rates of 60.0 to 84.8% against naive RAG and 49.6 to 54.8% against GraphRAG on four UltraDomain sets, judged by GPT-4o-mini ([arXiv 2410.05779](https://arxiv.org/abs/2410.05779), Table 1).

These win rates do not hold up under controlled judging:

- Zeng et al. found that swapping answer order changes win rates by over 30%, and that a 25-token length gap can shift them by over 50%. After removing both biases, "NaiveRAG slightly outperforms LightRAG". On the Agriculture set, LightRAG's win rate fell from 66.70% to 39.06% ([arXiv 2506.06331](https://arxiv.org/abs/2506.06331)).
- Han et al. saw GraphRAG global win 0.91 when shown first and 0.09 when shown second on the same QMSum comparison. After averaging both orders, "RAG is consistently preferred in terms of Comprehensiveness, while GraphRAG (Global) is favored for Diversity". On reference-based ROUGE-2, GraphRAG global was the worst method on all four summary sets ([arXiv 2502.11371](https://arxiv.org/abs/2502.11371)).
- GraphRAG-Bench, with accuracy scoring instead of pairwise preference, has GraphRAG local at 64.40 and HippoRAG 2 at 64.10 against 51.30 for plain RAG with reranking on contextual summarization ([arXiv 2506.05690](https://arxiv.org/abs/2506.05690)). This is the one independent result in GraphRAG's favor on summaries.

Microsoft's LazyGraphRAG claims indexing at "0.1% of the costs of full GraphRAG" and answer quality comparable to GraphRAG global search, judged by an LLM on 5,590 news articles ([Microsoft Research blog, 2024-11-25](https://www.microsoft.com/en-us/research/blog/lazygraphrag-setting-a-new-standard-for-quality-and-cost/)). The open-source library does not include it; the blog says it went into Microsoft Discovery and Azure Local. GraphRAG-Bench measured a LazyGraphRAG implementation at 591,150 indexing tokens for one 56k-token book, about the same as full GraphRAG's 654,673, so its cost claim is not confirmed.

In a project memory, summary questions usually target a small, filterable set: the workshop results or the decision records. Thunderbird's 128 Markdown files are 0.9 MB, about 230,000 tokens. A query-time map-reduce over that set in 32k-token batches is about 10 calls: 106 seconds of prefill at 2,165 tokens per second and about 55 seconds of generation for 500-token partial summaries at 91 tokens per second. That is about 3 minutes per question with no index-time cost. This is an estimate from the speeds in the [local evaluation research](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md).

### Temporal questions: which decision is current

Zep's Graphiti stores facts with validity windows. A newer contradicting fact sets the older fact's `invalid_at`, and the older fact stays in the graph ([arXiv 2501.13956](https://arxiv.org/abs/2501.13956)). On LongMemEval, the authors report:

- Temporal reasoning: 54.1% against 36.5% for full context with gpt-4o-mini, and 62.4% against 45.1% with gpt-4o.
- Knowledge update, the closest type to "which decision is current": 74.4% against 76.9% with gpt-4o-mini, and 83.3% against 78.2% with gpt-4o. Zep loses or gains 5 points.
- Single-session assistant questions: Zep loses 6.8 and 14.2 points.

The baseline in these numbers is the full conversation in context, not a retrieval system. The only third-party numbers come from Mem0, a competing product: on LoCoMo, Zep scored 49.31 on temporal questions against 55.51 for Mem0 and 58.13 for Mem0's graph variant, and full context scored 72.90 overall, above every memory system. Mem0 also reports that Zep's graph exceeded 600,000 tokens for a 26,000-token conversation, and that answers right after ingestion were worse than hours later ([arXiv 2504.19413](https://arxiv.org/abs/2504.19413)). Neither source is neutral.

Han et al. give an independent signal that structure helps temporal questions: on MultiHop-RAG temporal queries, GraphRAG global scored 53.34 and local 50.60 against 30.70 for plain RAG ([arXiv 2502.11371](https://arxiv.org/abs/2502.11371)). Those queries ask about the order of news events, which an LLM must infer from prose.

Decision records are different. By the glossary's definition, a decision record is "dated" and has "its status and the decision that supersedes it, if any". When the format chosen in [issue #9](https://github.com/Niko4417/OPT-cc/issues/9) carries these as front matter, the ingestion job reads the supersession chain directly and the answer prompt can mark superseded records. This needs no extraction model and cannot invent a false supersession. Graphiti's model fits workshop results better, where a later workshop overturns an earlier statement without a formal record, but its costs (section "Tools") outweigh that case for V1.

### Code dependency questions

Code graphs from static analysis are a different kind of graph. A parser builds them, and no LLM reads the code at index time:

- Code property graphs merge the abstract syntax tree, control flow graph and program dependence graph into one queryable graph (Yamaguchi et al., "Modeling and Discovering Vulnerabilities with Code Property Graphs", IEEE S&P 2014, [doi:10.1109/SP.2014.44](https://doi.org/10.1109/SP.2014.44)). Joern, the maintained implementation, has frontends for Java, Kotlin and ABAP among 14 others ([joern-cli/frontends](https://github.com/joernio/joern/tree/master/joern-cli/frontends)).
- RepoGraph builds a line-level graph of definitions and references with tree-sitter and no LLM calls. It raised SWE-bench Lite resolve rates from 27.33% to 29.67% for Agentless with GPT-4o and from 19.00% to 21.33% for AutoCodeRover with GPT-4 ([arXiv 2410.14684](https://arxiv.org/abs/2410.14684), ICLR 2025, Table 2). These are the authors' numbers.
- LocAgent parses code into a graph of files, classes and functions with import, call and inheritance edges, and reports 92.7% file-level localization and 86% lower cost with a fine-tuned 32B model ([arXiv 2503.09089](https://arxiv.org/abs/2503.09089)). Authors' numbers.
- CodexGraph lets an agent query a code graph database and reports "competitive performance" on CrossCodeEval, SWE-bench and EvoCodeBench ([arXiv 2408.03910](https://arxiv.org/abs/2408.03910)). Authors' claim.

Independent evidence on question answering is thin and Java-specific:

- StackRepoQA tested Claude 3.5 Sonnet on 1,318 Stack Overflow questions from 134 Java projects. A Neo4j graph of classes, methods and fields with CALLS, CONTAINS, EXTENDS and USES edges scored 6.12 against 5.66 for file-level retrieval on repository-related questions, on a 1 to 10 judge scale. Plain prompting without retrieval scored 6.54, which the authors trace to memorized Stack Overflow answers: scores dropped by about 1 point on questions after the model's cutoff ([arXiv 2603.26567](https://arxiv.org/abs/2603.26567)).
- SWE-QA (576 questions over 12 Python repositories, ACL 2026 Findings) did not test a graph retriever. With Qwen3-Coder-30B-A3B, a sibling of the lab's answer model, sliding-window RAG raised the score from 50.80 to 64.86 out of 100, and the OpenHands agent reached 65.88. "What" questions about architecture were the hardest type ([arXiv 2509.14635](https://arxiv.org/abs/2509.14635), Tables 3 and 4).

The evidence supports a static code graph as a tool for dependency and impact questions, where chunk retrieval cannot see call edges. It does not show that a code graph raises general answer quality.

## Indexing cost on the 64 GB Mac

No study used the lab models, so this section converts published token counts into local hours.

Inputs:

- Corpus, from the [test dataset research](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md): 12.4 MB of Kotlin and Java, 0.9 MB of Markdown, and about 5.7 MB of issue and pull request bodies since 2024 without comments. At about 3.5 characters per token for code and 4 for prose, that is about 5.2 million tokens. Without code it is 1.65 million, and the Markdown alone is 0.23 million. The characters-per-token figures are estimates.
- Speeds for `mlx-community/Qwen3.6-35B-A3B-4bit` on an M5 Pro: 2,165 tokens per second prefill and 91.3 tokens per second generation at 4k context, one request at a time ([oMLX benchmark](https://omlx.ai/benchmarks/mrpdgltl), via the [local evaluation research](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md)). The lab runs `mlx_lm.server` with `--decode-concurrency 1` ([model server research](https://github.com/Niko4417/OPT-cc/blob/research/model-server/docs/research/model-server.md)).
- Tokens per corpus token, from GraphRAG-Bench's indexing of one 56k-token book with GPT-4o-mini ([arXiv 2506.05690](https://arxiv.org/abs/2506.05690), Table 10): LightRAG 7.2 input and 1.26 output, HippoRAG 2 5.1 and 0.83, Microsoft GraphRAG 9.6 and 2.12, RAPTOR 2.0 and 0.03.

Estimated local indexing time, generation-bound:

| Method | Full repo, 5.2M tokens | Docs, issues and PRs, 1.65M tokens | Markdown only, 0.23M tokens |
|---|---|---|---|
| Microsoft GraphRAG (standard) | 40 h (50M in, 11M out) | 12.7 h | 1.7 h |
| LightRAG | 25 h (37M in, 6.6M out) | 7.9 h | 1.1 h |
| HippoRAG 2 | 17 h (26M in, 4.3M out) | 5.2 h | 0.7 h |
| RAPTOR | 2 h (10.5M in, 0.2M out) | 0.6 h | 0.1 h |
| Static code graph, link graph | minutes, 0 LLM calls | seconds, 0 LLM calls | seconds |

Checks and caveats:

- Generation is 75 to 85% of each LLM-graph total, so these figures scale with how verbose the model's extraction output is. Qwen3.6's thinking must be off for extraction. LightRAG's own config warns that a thinking model can spend its whole budget on hidden reasoning and return empty entities ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)), and the local evaluation research saw about fivefold output growth with Gemma's thinking on.
- LightRAG chunks at 1,200 tokens with 100 overlap and runs one gleaning pass by default (`DEFAULT_MAX_GLEANING = 1` in [constants.py](https://github.com/HKUDS/LightRAG/blob/main/lightrag/constants.py)). The full repo is about 4,700 chunks, so about 9,400 extraction calls plus merge summaries when an entity collects 8 or more description fragments. At about 20 seconds per chunk that matches the 25-hour row.
- In the HippoRAG 2 paper's own run on MuSiQue (11,656 passages, Llama-3.3-70B), indexing took 12 minutes for embeddings only, 100 minutes for HippoRAG 2, 235 minutes for LightRAG and 277 minutes for GraphRAG, with 9.2M, 68.5M and 115.5M input tokens for the last three ([arXiv 2502.14802](https://arxiv.org/abs/2502.14802), Table 12). The ratios between methods match the table above.
- `--decode-concurrency` above 1 may raise total throughput on the MoE model. Nobody has measured it with the lab pair, and a parallel extraction run competes with the checker model for memory.
- The default LightRAG entity types are prose types such as Person, Organization and Location ([entity type sample](https://github.com/HKUDS/LightRAG/blob/main/prompts/samples/entity_type_prompt.sample.yml)). Code chunks would need a custom profile, and the number of entities per code chunk, capped at 40 by `MAX_EXTRACTION_ENTITIES`, is unknown.

Query-time cost also differs. GraphRAG-Bench measured average prompts of 879 tokens for plain RAG, 1,008 for HippoRAG 2, 38,707 for GraphRAG local, 100,832 for LightRAG and 331,375 for GraphRAG global ([arXiv 2506.05690](https://arxiv.org/abs/2506.05690)). The current LightRAG default caps the prompt at `MAX_TOTAL_TOKENS=30000` ([env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). The memory budget from ticket #4 limits every route to 32k tokens, so LightRAG's cap should drop to about 24k to leave room for its system prompt.

## Incremental updates

| Method | Add a document | Change or delete a document | Source |
|---|---|---|---|
| LightRAG | Merges new entities and relations without a rebuild | Deletes by document ID and rebuilds affected entities from the remaining chunks (`rebuild_knowledge_from_chunks`); a changed file is delete plus insert at about 20 s per chunk | [lightrag.py](https://github.com/HKUDS/LightRAG/blob/main/lightrag/lightrag.py), [LightRAG paper](https://arxiv.org/abs/2410.05779) |
| Microsoft GraphRAG | `graphrag update` writes a new index to `update_output`; community reports regenerate | Same command | [cli/main.py](https://github.com/microsoft/graphrag/blob/main/packages/graphrag/graphrag/cli/main.py) |
| HippoRAG 2 | Skips passages already indexed | Deletes by text or `source_id` in 2.0.0a5 (repository); PyPI has 2.0.0a4 of 2025-06-24 | [README](https://github.com/OSU-NLP-Group/HippoRAG) |
| Graphiti | Ingests episodes one at a time per group by default; concurrent ingestion "can be different from the graph of serial ingestion" | Invalidates facts, does not delete them | [README](https://github.com/getzep/graphiti) |
| Static code graph | Re-parses changed files; codebase-memory-mcp runs a file watcher | Same | [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) |
| Link graph from metadata | Re-reads front matter and references | Same | Glue code |

Graphiti's invalidate-only model conflicts with deleting content from exported copies, a gap the map already lists. The [stack landscape research](https://github.com/Niko4417/OPT-cc/blob/research/rag-stack-landscape/docs/research/rag-stack-landscape.md) notes it needs a hard-delete path for GDPR erasure.

## Tools

| Tool | License | Latest release | Graph store | Local-model notes | Fit |
|---|---|---|---|---|---|
| [HKUDS/LightRAG](https://github.com/HKUDS/LightRAG) | MIT | v1.5.8rc1, 2026-11-15 | Local files, PostgreSQL, Neo4j | OpenAI-compatible bindings; thinking off for extraction | Already shortlisted |
| [OSU-NLP-Group/HippoRAG](https://github.com/OSU-NLP-Group/HippoRAG) | MIT | 2.0.0a4 on PyPI, 2025-06-24; repository pushed 2026-10-06 | igraph plus Parquet or Milvus Lite | OpenAI-compatible LLM and embedding endpoints. The example embedder NV-Embed-v2 is [CC-BY-NC-4.0](https://huggingface.co/nvidia/NV-Embed-v2), so use Qwen3-Embedding instead. Pins `torch==2.5.1`; vLLM is an optional extra | Retrieval-only arm |
| [microsoft/graphrag](https://github.com/microsoft/graphrag) | MIT | v3.3.0, 2026-10-08 | Parquet files | Docs report malformed JSON from proxied local models ([model config](https://microsoft.github.io/graphrag/config/models/)). FastGraphRAG replaces LLM extraction with NLTK or spaCy noun phrases but still writes community reports with the LLM ([methods](https://microsoft.github.io/graphrag/index/methods/)) | No |
| [OpenSPG/KAG](https://github.com/OpenSPG/KAG) | Apache-2.0 | v0.8.0, 2025-06-28 | OpenSPG server through Docker Compose | No local-model path in the README; claims an 89% token cut in "Lightweight Build" without method | No |
| [parthsarthi03/raptor](https://github.com/parthsarthi03/raptor) | MIT | No release; last push 2024-09-03 | In memory | Unmaintained | No |
| [getzep/graphiti](https://github.com/getzep/graphiti) | Apache-2.0 | v0.31.2, 2026-10-20 | Neo4j 5.26 (GPL-3.0 Community), FalkorDB (SSPL), FalkorDB Lite, Neptune | README: "Very small models frequently emit JSON that doesn't match the requested schema" | Later, for workshop facts |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | v1.6.8, 2026-11-12 | Several | Can run with GLiNER instead of an LLM | No published QA evidence found |
| [DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) | MIT | v0.12.1, 2026-10-16 | SQLite | No generative model; tree-sitter plus type resolution for Java and Kotlin; claims a Linux kernel index in about 3 minutes on an M3 Pro | Code graph candidate |
| [vitali87/code-graph-rag](https://github.com/vitali87/code-graph-rag) | MIT | v0.7.1, 2026-11-17 | SQLite, Memgraph or Neo4j | Java fully supported, Kotlin "structural support" only | Code graph candidate |
| [CodeGraphContext](https://github.com/CodeGraphContext/CodeGraphContext) | MIT | v0.5.7, 2026-08-08 | Graph database | Tree-sitter and SCIP indexers | Code graph candidate |
| [joernio/joern](https://github.com/joernio/joern) | Apache-2.0 | v4.0.685, 2026-11-17 | Own code property graph | Frontends for Java, Kotlin, ABAP and others | Code graph for legacy languages |
| [scip-code/scip-java](https://github.com/scip-code/scip-java) | Apache-2.0 | v0.14.2, 2026-11-12 | SCIP index files | Needs a working build | Precise Java cross-references |

Two stores the other tools used are gone: [kuzudb/kuzu](https://github.com/kuzudb/kuzu) was archived on 2025-10-10 and [github/stack-graphs](https://github.com/github/stack-graphs) on 2025-09-09. Neither is safe to build on.

## Recommendation

1. **No LLM-extracted graph in the base stack.** It loses on single-fact questions and refusals, which most project memory questions are. It gains 1 to 13 points on multi-hop and some summary questions in independent tests, and one indexing of the test dataset costs 5 to 40 hours of local generation, depending on method and scope. Its published wins rely on biased pairwise judging.
2. **A link graph as part of the glue ingestion job.** Nodes are decision records, workshop results, issues, pull requests and files. Edges come from front matter (`status`, `supersedes`, date), from references the job can parse (`#1234`, ADR and RFC numbers, URLs, file paths) and from the pull request file lists the GitHub API returns (ticket #5's ingestion job already fetches pull requests). Retrieval expands a hit one or two hops along these edges, and the answer prompt marks superseded decision records. This answers "which decision is current" and "decision to issue to code" without LLM calls. It belongs in the format grilling of [issue #9](https://github.com/Niko4417/OPT-cc/issues/9): `status`, `date` and `supersedes` must be machine-readable.
3. **A static code graph exposed as a tool for dependency questions.** codebase-memory-mcp is the first one to test: MIT, SQLite, no generative model, Java and Kotlin type resolution, a file watcher. Joern is the fallback for languages without tree-sitter tag queries; it already has an ABAP frontend, which the map lists as unspecified.
4. **Global summaries by query-time map-reduce** over the filtered document set, about 3 minutes per question on the test dataset's Markdown, instead of precomputed community reports.

For the head-to-head in [issue #13](https://github.com/Niko4417/OPT-cc/issues/13):

- Keep LightRAG. Index only Markdown docs, issues and pull requests for it (about 8 hours), not the code, and record the indexing time. Turn Qwen3.6's thinking off for extraction, supply an entity type profile for decision records and modules, and set `MAX_TOTAL_TOKENS` to about 24k.
- Add HippoRAG 2 as a retrieval-only arm feeding the same answer prompt, indexed on the same subset (about 5 hours). It has the best independent record among LLM graphs (parity on simple questions, the top multi-hop and complex reasoning scores in two studies) and costs about two thirds of LightRAG. It has no UI, so it can only win as a retrieval component inside the winning app, not as a stack.
- Do not add Microsoft GraphRAG (40 hours, unreliable JSON from local models), KAG (Docker server, no local-model path), RAPTOR (unmaintained, behind plain RAG in three studies) or Graphiti (structured-output failures with small local models, GPL or SSPL store, serial ingestion).
- The gold question set in [issue #11](https://github.com/Niko4417/OPT-cc/issues/11) must tag each question by type: single fact, multi-hop, global summary, current decision, code dependency and unanswerable. The bake-off must report scores per type. Without that split, a graph's gain on 5 multi-hop questions disappears in the average of 50.

## Open points

- No study measured any LLM graph with Qwen3.6-35B-A3B or on code. The hours above are conversions from GPT-4o-mini token ratios and need a measured run.
- The gain from a link graph and a code graph is not measured by any paper. It rests on the structure of decision records and code. The bake-off can test it as an add-on to the winning stack.
- Graphiti may fit workshop results where a later workshop overturns an earlier statement without a formal decision record. Revisit it if the gold set shows the current-decision type failing with metadata alone.

## Sources

Papers:

- Edge et al., From Local to Global: A Graph RAG Approach to Query-Focused Summarization: https://arxiv.org/abs/2404.16130
- Guo et al., LightRAG: Simple and Fast Retrieval-Augmented Generation: https://arxiv.org/abs/2410.05779
- Gutiérrez et al., HippoRAG: https://arxiv.org/abs/2405.14831
- Gutiérrez et al., From RAG to Memory (HippoRAG 2): https://arxiv.org/abs/2502.14802
- Liang et al., KAG: https://arxiv.org/abs/2409.13731
- Sarthi et al., RAPTOR: https://arxiv.org/abs/2401.18059
- Rasmussen et al., Zep: A Temporal Knowledge Graph Architecture for Agent Memory: https://arxiv.org/abs/2501.13956
- Chhikara et al., Mem0: https://arxiv.org/abs/2504.19413
- Han et al., RAG vs. GraphRAG: A Systematic Evaluation and Key Insights: https://arxiv.org/abs/2502.11371
- Xiang et al., When to use Graphs in RAG (GraphRAG-Bench): https://arxiv.org/abs/2506.05690
- Zeng et al., How Significant Are the Real Performance Gains? An Unbiased Evaluation Framework for GraphRAG: https://arxiv.org/abs/2506.06331
- Microsoft Research, LazyGraphRAG blog: https://www.microsoft.com/en-us/research/blog/lazygraphrag-setting-a-new-standard-for-quality-and-cost/
- Yamaguchi et al., Modeling and Discovering Vulnerabilities with Code Property Graphs: https://doi.org/10.1109/SP.2014.44
- Ouyang et al., RepoGraph: https://arxiv.org/abs/2410.14684
- Chen et al., LocAgent: https://arxiv.org/abs/2503.09089
- Liu et al., CodexGraph: https://arxiv.org/abs/2408.03910
- Alebachew et al., Beyond Code Snippets (StackRepoQA): https://arxiv.org/abs/2603.26567
- Peng et al., SWE-QA: https://arxiv.org/abs/2509.14635

Tools and docs:

- LightRAG: https://github.com/HKUDS/LightRAG (env.example, lightrag/constants.py, lightrag/lightrag.py, prompts/samples/entity_type_prompt.sample.yml)
- Microsoft GraphRAG: https://github.com/microsoft/graphrag, https://microsoft.github.io/graphrag/index/methods/, https://microsoft.github.io/graphrag/config/models/
- HippoRAG: https://github.com/OSU-NLP-Group/HippoRAG, https://pypi.org/project/hipporag/
- KAG: https://github.com/OpenSPG/KAG
- RAPTOR: https://github.com/parthsarthi03/raptor
- Graphiti: https://github.com/getzep/graphiti
- Cognee: https://github.com/topoteretes/cognee
- codebase-memory-mcp: https://github.com/DeusData/codebase-memory-mcp
- code-graph-rag: https://github.com/vitali87/code-graph-rag
- CodeGraphContext: https://github.com/CodeGraphContext/CodeGraphContext
- Joern: https://github.com/joernio/joern
- scip-java: https://github.com/scip-code/scip-java
- NV-Embed-v2 model card: https://huggingface.co/nvidia/NV-Embed-v2

Earlier research in this repository: `docs/research/test-dataset.md`, `docs/research/local-evaluation.md`, `docs/research/model-server.md`, `docs/research/stack-scoring.md` and `docs/research/rag-stack-landscape.md` on their `research/*` branches.
