# How a project memory should retrieve code from large repositories

Date: 2026-11-19. Ticket: [#32](https://github.com/Niko4417/OPT-cc/issues/32). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

How should a project memory retrieve code from a repository that may be very large? The candidates are embedding retrieval over code chunks, agentic search (grep and file reads in a loop), repo maps, code graphs and hybrids. Are the Keiko team's three claims true? What do the approaches cost in indexing time, query latency, tokens and money, and how do they cope with repositories too large for one index or one context window? Which approach should join the bake-off in [#13](https://github.com/Niko4417/OPT-cc/issues/13), and what does it need from the models on-device and on a cloud provider?

## Answer

Use a hybrid. Build a lexical index and an embedding index over tree-sitter chunks, and give both to an agent as search tools next to grep and line-range file reads. Neither pure approach wins in the evidence.

- Embedding retrieval over code works for natural-language questions. In every benchmark found, the best dense code retriever ranks code above BM25, though not every dense model does ([CodeRAG-Bench](https://arxiv.org/abs/2406.14497), [LocAgent](https://arxiv.org/abs/2503.09089), [RepoPremium](https://arxiv.org/abs/2611.20455), [Agent Retrieval Bench](https://arxiv.org/abs/2607.24882)). The margin is smaller than older benchmarks suggest. BM25 over whole files, or BM25 fed with identifiers that a model writes into the query, closes most of it ([RepoPremium](https://arxiv.org/abs/2611.20455)). The fused result of both beats either one alone.
- Agentic search is not stronger by default. In the only controlled study of repository question answering found, an agent with an embedding search tool answered 65.2% of 720 questions correctly against 46.2% for a grep agent with subagents. It cost $0.32 per correct answer against $0.74 ([Oskooei et al. 2026](https://arxiv.org/abs/2608.01507)). On SWE-QA, the better one-shot RAG arm scored between 0 and 9.4 points below the better agent for each of six models, while sending 7 to 54 times fewer input tokens ([SWE-QA](https://arxiv.org/abs/2509.14635), Tables 3 and 8).
- The coding tools split. Claude Code and Codex CLI use grep and file reads with no index. Anthropic gives staleness, privacy and simplicity as reasons and has published no retrieval benchmark ([Anthropic, 2026-05-14](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start)). Cursor published a 12.5% average accuracy gain from adding semantic search to grep in November 2025 ([Cursor, 2025-11-06](https://cursor.com/blog/semsearch)). Cursor then wound its embedding index down in July 2026, saying current models grep well, without publishing new numbers ([Cursor forum, 2026-07](https://forum.cursor.com/t/what-do-you-think-about-cursor-removing-the-codebase-indexing-settings/165899)).
- A project memory differs from a coding agent in ways that favor an index. It answers questions about a repository pinned to a commit, so staleness is a re-index of changed blobs, not a race with live edits. It runs on a 32k-token on-device prompt budget, where an agent loop of 42k to 760k input tokens per question is slow. Its users ask conceptual questions such as "how does sync work", which do not name an identifier to grep for.

Verdicts on the Keiko team's claims:

| Claim | Verdict | Evidence |
|---|---|---|
| Embedding retrieval over code is "backwards" because code is not natural language | Contradicted | Dense retrievers find code for natural-language questions in every benchmark found. Keiko itself embeds repository chunks and fuses them with lexical hits (section "Keiko") |
| RAG is a one-shot search and ranking, while recursive search keeps searching until it finds the answer | Contradicted | Iterative RAG exists since RepoCoder (2023). Agents with an embedding search tool search many times. Keiko's own Chat search runs a fixed plan with at most 16 searches and at most 1 follow-up pass. Agents stop without touching any gold file on 27 to 35% of samples ([Agent Retrieval Bench](https://arxiv.org/abs/2607.24882)) |
| Keiko's recursive search reads files of up to 2 MB each | Partly confirmed | 2 MiB (2,097,152 bytes) is the per-file limit for scanning and excerpting. "Recursive" means a scan of the whole folder tree, not a model loop. The model sees 512-byte snippets in the coding tool and at most 128 KiB of excerpts per Chat turn, never whole 2 MB files |

For the bake-off, add three code-retrieval arms inside the winning stack: one-shot hybrid RAG, a grep-only agent and a hybrid agent (section "Recommendation for the bake-off").

## Evidence on embedding retrieval for code

### Natural-language-to-code retrieval works

The field has measured this since CodeSearchNet, which paired about 2 million functions with their documentation and added 99 natural-language queries with about 4,000 expert relevance labels ([Husain et al. 2019](https://arxiv.org/abs/1909.09436)). The repository was archived on 2023-04-11 ([github/CodeSearchNet](https://github.com/github/CodeSearchNet)). Newer benchmarks:

- CodeRAG-Bench measured retrieval on six task sets. On SWE-bench Lite, BM25 scored nDCG@10 43.0, Jina-v2-code 58.3 and SFR-Mistral 62.7. On RepoEval, BM25 scored 93.2 and Voyage-code 94.3. On DS-1000, BM25 scored 5.2 and Voyage-code 33.1 ([arXiv 2406.14497](https://arxiv.org/abs/2406.14497), Table 3). With retrieve-then-rerank, GPT-4o's SWE-bench pass@1 rose from 2.3 to 21.7, against 30.7 with gold context (Table 6). The authors also report that retrievers "struggle to fetch useful contexts especially with limited lexical overlap".
- LocAgent's baselines on SWE-bench Lite: the 137M-parameter CodeRankEmbed found a gold file in the top 5 for 84.67% of issues, against 61.68% for BM25. It beat Agentless with GPT-4o (74.45%) and SWE-agent with GPT-4o (68.98%), and lost to SWE-agent and OpenHands with Claude 3.5 (90.15%) ([arXiv 2503.09089](https://arxiv.org/abs/2503.09089) v2, Table 4).
- Agent Retrieval Bench (427 samples, 25 repositories in six languages) measured file recall@20 at 0.7029 for Qwen3-Embedding-8B, 0.6333 for a vectorless repo map, 0.4940 for a path-aware lexical heuristic and 0.4452 for BM25 ([arXiv 2607.24882](https://arxiv.org/abs/2607.24882), Table 4). Lexical search led all embeddings only on stack-trace-to-code questions.
- CORE-Bench found that the same embedding models drop sharply from function-level code search to repository tasks. Qwen3-Embedding-8B scored nDCG@10 71.7 on code understanding and 20.3 on issue-to-edit localization. Fine-tuning on in-domain data raised the latter to 32.8 ([arXiv 2606.11864](https://arxiv.org/abs/2606.11864) v3, Table 3). This is a limit of off-the-shelf models on hard queries, not proof that code cannot be embedded.

### The dense advantage is smaller than it looks

RepoPremium is the most careful test found. It ranks the files of 10 Python repositories for 1,265 change descriptions and adds a preregistered lockbox of 539 queries over 91 repositories ([Xue et al. 2026](https://arxiv.org/abs/2611.20455), v1 of 2026-11-16):

- With 1,000-character chunks, dense retrieval led BM25 by 0.121 nDCG@10. Indexing whole files for BM25 cut the lead to 0.034. On raw pull request text from 2022, dense still led 0.572 to 0.447.
- When a model first wrote likely identifiers into the query ("generate-then-grep"), BM25 gained 0.11 to 0.19 and was never behind dense in point estimates. Dense gained only 0.03 to 0.07 from the same expansion. Prose phrases did not help. Identifiers, API names and error strings did.
- Fusion won. On the lockbox, BM25 scored 0.432, dense 0.533, reciprocal rank fusion of both 0.551, and fusion of dense with generate-then-grep 0.578. A per-query router did not beat fusion.
- With the LLM fixed, a SWE-agent's localization stayed within about 0.04 across grep, a lexical index and a dense index. The authors conclude that for agents, the choice of LLM matters more than the search tool.

The study is 3 days old and has not been reviewed. Its guidelines (fair lexical baseline, keep dense and fuse it with BM25, expand queries with identifiers) match the other benchmarks.

## Evidence on agentic and iterative search

### Iterative retrieval is a RAG technique

RepoCoder alternates retrieval and generation, using the draft completion as the next query. It beat in-file completion "by over 10% in all settings" and beat single-pass retrieval ([Zhang et al., EMNLP 2023](https://arxiv.org/abs/2303.12570)). Agentless localizes with a model that reads the repository tree and with embedding retrieval over chunks of the files it did not rule out, then keeps the files both agree on. The combination contained the gold file for 81.67% of issues, against 78.67% for the model alone and 70.33% for embeddings alone ([Xia et al.](https://arxiv.org/abs/2407.01489) v2, Table 2). A RAG pipeline can therefore search several times, and an agent can call a vector index as one of its tools. The split between "one-shot RAG" and "search until found" describes two configurations, not two methods.

### Controlled comparisons

| Study | Arms | Accuracy | Tokens and cost | Source |
|---|---|---|---|---|
| SWE-QA, 576 questions, 12 Python repositories | Direct prompt, chunk RAG, sliding-window RAG, SWE-agent, OpenHands, six models | Qwen3-Coder-30B-A3B total score out of 100: direct 50.80, sliding-window RAG 64.86, SWE-agent 57.44, OpenHands 65.88. GPT-5.1: 61.41, 66.79, 69.20, 70.79. Cursor's agent scored 70.66 | Input tokens per question: RAG 3,042 to 6,302; agents 42,433 to 165,206 | [arXiv 2509.14635](https://arxiv.org/abs/2509.14635) v2, Tables 3 and 8 |
| Oskooei et al., SWE-QA questions, 15 repositories of 13k to 861k lines, 720 questions per arm | ReAct agent with `search_rag` (text-embedding-3-small, Chroma, 500-token chunks, top 10) against a planner agent with grep, glob, read and subagents, four models | Pass rate 65.2% against 46.2%. The semantic arm won with every model, by 5.6 to 49.6 points | $0.32 against $0.74 per correct answer. Input tokens 34k to 404k against 466k to 761k per question. Embedding all 15 repositories (21M tokens) cost $0.43 | [arXiv 2608.01507](https://arxiv.org/abs/2608.01507), Table 3, Appendix B |
| Agent Retrieval Bench, logged agent trajectories | OpenAI GPT-5.4-mini agent, Codex CLI with GPT-5.4 and GPT-5.5 | Agents touched no gold file on 35.2% (OpenAI) and 27.2 to 29.3% (Codex) of samples | Seeding Codex with 8 lexical or fused retrieval hits raised final file F1 from 0.32 to about 0.40, not statistically resolved | [arXiv 2607.24882](https://arxiv.org/abs/2607.24882), Tables 13 and 16 |
| Cursor Context Bench (Cursor's own) | Agent with grep against agent with grep and semantic search, all models in Cursor | 12.5% higher accuracy on average, 6.5 to 23.5% by model. Online A/B test: code retention +0.3%, +2.6% on codebases with 1,000+ files | Not published | [Cursor blog, 2025-11-06](https://cursor.com/blog/semsearch) |

Caveats:

- Oskooei et al. confound two variables. The losing arm also delegates to subagents, and 41.8% of its failures were hand-off failures between planner and subagent. The paper did not test a single grep agent without subagents. It shows that a vector tool helps an agent, not that grep alone fails.
- SWE-QA's RAG arms sent 3,042 to 6,302 input tokens per question, which fits the 32k on-device budget. Its agents needed 42k to 165k.
- Cursor's numbers are the vendor's own, on an unpublished benchmark, with an embedding model trained on Cursor's agent traces.

### What the coding tools do

| Tool | Code retrieval | Evidence of quality | Source |
|---|---|---|---|
| Claude Code | No index. Grep, glob, file reads and optional LSP. Anthropic recommends CLAUDE.md files, LSP and read-only subagents for large codebases | No benchmark published. A May 2025 report quotes Boris Cherny that the comparison with RAG was "mostly vibes", with some internal benchmarks | [Anthropic, 2026-05-14](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start), [Agent SDK post, 2025-09-29](https://claude.com/blog/building-agents-with-the-claude-agent-sdk), [OfficeChai report, 2025-05-27](https://officechai.com/ai/claude-researcher-explains-how-agentic-search-performed-better-than-rag-for-code-generation/) (secondary) |
| Codex CLI | No index. The system prompt tells the model to use `rg` and `rg --files`. The source tree has no embedding code | Logged Codex trajectories missed every gold file on 27 to 29% of samples (above) | [`gpt_5_codex_prompt.md` at `90dd6cf9`](https://github.com/openai/codex/blob/90dd6cf9df1d3236a7f8085f65eb76c75277646f/codex-rs/core/gpt_5_codex_prompt.md) |
| Cursor | Grep and a custom embedding index until mid-2026. Now a local trigram index ("Instant Grep") and an Explore subagent. The docs say Cursor "does not store embeddings of your codebase for search" | 12.5% gain from embeddings in 2025 (above). Staff wrote in July 2026 that the semantic path was "no longer helping in a meaningful way", with no published numbers | [semantic search docs](https://cursor.com/docs/context/semantic-search), [fast regex search, 2026-03-23](https://cursor.com/blog/fast-regex-search), [forum thread](https://forum.cursor.com/t/what-do-you-think-about-cursor-removing-the-codebase-indexing-settings/165899) |
| Sourcegraph Cody | Replaced embeddings with Sourcegraph's keyword search in 2024 | Sourcegraph's FAQ: "Early evidence suggests that this context fetching works as well as embeddings and is sometimes better". No data shown | [Sourcegraph blog, 2024-02-15](https://sourcegraph.com/blog/how-cody-understands-your-codebase), [Cody FAQ 5.7](https://5.7.sourcegraph.com/cody/faq) |
| Continue | Deprecated `@Codebase`, which combined local transformers.js embeddings, keyword search and LLM reranking (25 candidates, 5 kept). Agent mode with file and search tools replaces it | No comparison published | [deprecated `@Codebase` page](https://docs.continue.dev/reference/deprecated-codebase) |
| Aider | Repo map: tree-sitter tags, a file dependency graph ranked by personalized PageRank, cut to `--map-tokens` (default 1,024 tokens) | A vectorless repo-map baseline of the same kind scored recall@20 0.6333 in Agent Retrieval Bench | [repomap docs](https://aider.chat/docs/repomap.html), [`repomap.py`](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/repomap.py) |
| Amp | Not verified. The manual does not describe how Amp searches a local repository | None found | [Amp manual](https://ampcode.com/manual) |

The tools' reasons for dropping embeddings are operational: index staleness while thousands of engineers commit, uploading code to an embedding service, admin work, and index size at more than 100,000 repositories (Sourcegraph). None of them published a benchmark where grep alone beats grep plus embeddings. Anthropic names the trade-off itself: semantic search "is usually faster than agentic search, but less accurate, more difficult to maintain, and less transparent", and agentic search "works best when Claude has enough starting context to know where to look".

## Keiko

I read `oscharko-dev/Keiko` on branch `dev` at commit [`bc57815c`](https://github.com/oscharko-dev/Keiko/commit/bc57815cbb383faa6531dd416afa5fe6ccd3d56a) of 2026-11-19 through `gh api`. Links below point at that commit. No published Keiko evaluation compares its recursive search with embedding retrieval.

What "recursive search" is:

- The constant is `MAX_RECURSIVE_TEXT_FILE_BYTES = 2_097_152`, commented "Inclusive source-file eligibility shared by recursive search, text preview and extraction" ([workspace-contract-primitives.ts L7-8](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-contracts/src/workspace-contract-primitives.ts#L7-L8)). Files over 2 MiB are skipped, not read.
- ADR-0175 defines repository search v1 as "local lexical/literal/safe-regex/symbol search". It caps queries at 200 characters, hits at 50, snippets at 512 bytes and the result at 64 KiB. Discovery imposes no file-count, depth or time ceiling on the folder ([ADR-0175 L300-319](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/adr/ADR-0175-canonical-governed-tool-catalog.md#L300-L319), [coding-repository-search.ts L9-21](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-contracts/src/coding-repository-search.ts#L9-L21)). "Recursive" therefore means a walk of every directory, as in `grep -r`.
- The scanner is pure JavaScript, "no subprocess, no ripgrep" ([repoSearch.ts L7](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-workspace/src/repoSearch.ts#L7)).
- In the Coding Workbench, an OpenCode agent calls this search as a tool and "refines a truncated search with a more selective query" ([ADR-0175 L341-344](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/adr/ADR-0175-canonical-governed-tool-catalog.md#L341-L344)). That part is agentic search.

How Keiko's Chat answers a question about a connected folder:

- A deterministic planner builds the plan "BEFORE any retrieval work runs", with no model call ([plan.ts L1-5](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-workflows/src/planner/plan.ts#L1-L5)). It composes up to three rings: a lexical scan (weight 0.55), structural lookups around identifiers and paths (0.30) and a git-history recency signal (0.15) ([plan.ts L113-117](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-workflows/src/planner/plan.ts#L113-L117)).
- The default budget per turn is 16 search calls, 131,072 bytes of excerpts, 116,000 model input tokens, 4,096 output tokens, 1 rerank call and 1 follow-up pass ([connected-context.ts L189-198](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-contracts/src/connected-context.ts#L189-L198)). One excerpt window is at most 8,192 bytes ([grounded-orchestrator.ts L3624](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-server/src/grounded-orchestrator.ts#L3624)).
- The follow-up pass runs only when the model declares the evidence insufficient. It re-retrieves at most 3 named unread paths, and then the budget's follow-up count is 0 ([grounded-follow-up.ts L62-91](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-server/src/grounded-follow-up.ts#L62-L91)). The Chat path does not keep searching until it finds the answer.
- Lexical candidates can be reranked by embeddings. Keiko reads vectors from the folder's repository Knowledge Pod, fuses lexical and semantic ranks by reciprocal rank fusion with k = 60, and falls back to lexical when the pod is missing or stale ([repoSearchSemantic.ts L13-18](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-workspace/src/repoSearchSemantic.ts#L13-L18), [grounded-repo-semantic-search.ts L742-744](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-server/src/grounded-repo-semantic-search.ts#L742-L744)). The repository pod chunks code at regex symbol boundaries ([GitHub ingestion research](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md)).
- Keiko's own retrieval epic sets the target that "Vector retrieval remains effective for semantic, multilingual, and natural-language questions" next to lexical retrieval for identifiers and error strings ([hybrid-retrieval-quality-ledger.md L52](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/local-knowledge/hybrid-retrieval-quality-ledger.md)).

Keiko's code is a lexical-first hybrid with a bounded plan and optional embedding rerank. It agrees with the evidence above more than the team's spoken claims do. The [stack scoring research](https://github.com/Niko4417/OPT-cc/blob/research/stack-scoring/docs/research/stack-scoring.md) notes open Keiko work (epic #4429) on a slow repeat search and a wrong-file bug in connected-folder search, so the bake-off must measure a build that includes it.

## Performance

No study measured these approaches with the lab models on the test dataset. The figures below combine published token counts with the lab speeds and are estimates unless marked as measured.

Inputs:

- Test dataset: `thunderbird/thunderbird-android` has 12.4 MB of Kotlin and Java, about 3.5 million tokens at 3.5 characters per token ([test dataset research](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md), [graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md)).
- On-device speeds for Qwen3.6-35B-A3B 4-bit on an M5 Pro: 2,165 tokens per second prefill and 91.3 tokens per second generation at 4k context ([oMLX benchmark](https://omlx.ai/benchmarks/mrpdgltl), via the [local evaluation research](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md)). Prefill is slower at 32k context.
- The memory budget caps every on-device prompt at 32k tokens ([memory fit research](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md)).

| Approach | Indexing, test dataset | Query latency | Tokens per question | Cloud cost per question |
|---|---|---|---|---|
| One-shot hybrid RAG (BM25 plus embeddings, rerank) | BM25: seconds. Embeddings with Qwen3-Embedding-0.6B on-device: at most about 27 minutes (3.5M tokens at 2,165 tokens per second, an upper bound because the 0.6B embedder computes about one fifth of the 3B active parameters per token). Cloud: about $0.07 at the $0.43 per 21M tokens that Oskooei et al. paid | Search 0.02 to 47 ms per query ([CodeRAG-Bench](https://arxiv.org/abs/2406.14497), Table 4). Answer on-device about 8 s, estimated in the local evaluation research for a document RAG prompt | 3,042 to 6,302 input, under 250 output ([SWE-QA](https://arxiv.org/abs/2509.14635), Table 8) | Not reported per question; tokens are 7 to 54 times fewer than an agent's |
| Agent with grep and file reads, no index | None | On-device: about 80 s or more for OpenHands' 115,879 input and 2,366 output tokens with Qwen3-Coder-30B-A3B, if every turn re-reads its prompt without a cache. Cloud, planner with subagents: median 105 to 262 s ([Oskooei et al.](https://arxiv.org/abs/2608.01507)) | 42,433 to 165,206 input on SWE-QA. 466k to 761k for the planner with subagents | Planner with subagents: $0.17 to $0.67 ([Oskooei et al.](https://arxiv.org/abs/2608.01507), Table 3) |
| Agent with an embedding search tool | As one-shot RAG | Cloud: median 42 to 400 s | 34k to 404k input | $0.0085 to $0.57 |
| Repo map or static code graph | Tree-sitter parse: seconds to minutes, no model calls ([graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md)) | Milliseconds per lookup | Aider's map is 1,024 tokens by default | Only the answer call |

The SWE-QA agent token counts are cumulative over all turns. An on-device agent must also keep each turn under the 32k-token prompt cap, so it needs a harness that drops old tool output or hands searches to a subagent. Gemma-4-31B, the lab's checker model, generates 13 tokens per second, so an agent loop on it would take about 7 times longer than on Qwen3.6.

### Repositories too large for one index or one context window

- Embedding indexes grow with the code and must be refreshed. Cursor reported that naive indexing of tens of thousands of files "can take hours", with a 99th-percentile time to first query of 4.03 hours. Reusing a teammate's index (clones were 92% similar) and caching embeddings by chunk content cut it to 21 seconds ([Cursor, 2026-01-27](https://cursor.com/blog/secure-codebase-indexing)). Sourcegraph found vector search over more than 100,000 repositories "complex and resource-intensive" ([Sourcegraph, 2024-02-15](https://sourcegraph.com/blog/how-cody-understands-your-codebase)). A project memory indexes one modernization project's repositories at pinned commits. It can update by git blob ID, which Keiko already does, so only changed files are re-embedded.
- Agentic search needs no index, but grep slows down and floods the context. Cursor saw `rg` calls of more than 15 seconds on large monorepos and built a trigram index for them ([Cursor, 2026-03-23](https://cursor.com/blog/fast-regex-search)). Anthropic writes that grep for a common function name returns thousands of matches, that LSP filters them by symbol, and that a vague search across a billion-line codebase hits context-window limits ([Anthropic, 2026-05-14](https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start)). Keiko's scanner has no file-count limit and its coding tool returns at most 50 hits, so on a large tree a broad query is truncated, and the agent must narrow it.
- Neither approach puts a whole large repository in context. RAG sends the top chunks. An agent reads a few files per turn and depends on subagents or summaries to stay inside the window. A repo map or code graph gives the agent a compact starting point either way.

## Recommendation for the bake-off

Add these arms to [#13](https://github.com/Niko4417/OPT-cc/issues/13). Run them inside the winning stack on the gold questions from [#11](https://github.com/Niko4417/OPT-cc/issues/11), against the test dataset at a pinned commit:

1. One-shot hybrid RAG. Tree-sitter chunks from the glue ingestion job in the [GitHub ingestion research](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md). BM25 over whole files and chunks, Qwen3-Embedding-0.6B over chunks, reciprocal rank fusion, Qwen3-Reranker-0.6B, top chunks into the answer prompt. This is the cheap baseline and needs no tool calling.
2. Grep-only agent. Tools: `rg` over a clone at the pinned commit, a file read with a line range, and a directory listing. This arm tests the Keiko team's claim directly. Keiko's own connected-folder search counts as a variant of it if Keiko is still in the bake-off.
3. Hybrid agent. The tools of arm 2, plus the arm 1 search as a tool, plus the static code graph tool from the [graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md) for dependency questions. This is the expected winner and the recommended design if it wins.

For each arm, record answer score per question type, the share of gold files retrieved, input and output tokens, tool calls, invalid tool calls, wall time per question and index build time. Arm 3 must beat arm 1 by more than its extra latency and cost is worth to a consultant, which the owner decides after the numbers.

What the models need:

| Location | Arm 1 | Arms 2 and 3 |
|---|---|---|
| On-device (Mac, 32k prompt cap) | No tool calling. 8k to 16k prompts. An embedding and a reranker model next to the answer model, about 2.4 GB of weights ([memory fit research](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md)) | Reliable multi-turn tool calling with valid JSON arguments from Qwen3.6-35B-A3B. No source measured that for the lab model, so the bake-off counts invalid calls. A harness that keeps each turn under 32k tokens. Expect one to several minutes per question |
| One cloud provider (ADR-0009) | Any chat model and the provider's embedding model. The provider's managed retrieval layer is an arm 1 variant under ADR-0009 | Tool calling and at least 128k context so that the loop needs no compaction. Prompt caching cuts the repeated prefix cost. Budget $0.01 to $0.70 per question from the published runs |

## Open points

- No study measured any of the arms with the lab models or on Kotlin and Java. The latency and indexing figures above are conversions and need a measured run.
- The only controlled repository QA comparison (Oskooei et al.) confounds subagents with grep, and RepoPremium is a few days old and unreviewed.
- Cursor wound down its embedding index in July 2026 after publishing a gain in November 2025. It has not published the evaluation behind the change. If it does, re-read it.
- Anthropic and OpenAI have published no retrieval benchmark for Claude Code or Codex CLI. Their design choice is evidence of what works for live code editing, not of what answers questions best over a pinned repository.

## Method

- Read the ticket, the glossary, ADR-0001 to ADR-0011 on `main`, and the GitHub ingestion, graph knowledge, memory fit, local evaluation and stack scoring research on their `research/*` branches.
- Read Keiko source through `gh api repos/oscharko-dev/Keiko/contents/...` at `bc57815c` and searched it with `gh search code`. Files: `repoSearch.ts`, `repoSearchSemantic.ts`, `workspace-contract-primitives.ts`, `coding-repository-search.ts`, `connected-context.ts`, `plan.ts`, `grounded-orchestrator.ts`, `grounded-follow-up.ts`, `grounded-repo-semantic-search.ts`, ADR-0175 and the Local Knowledge quality ledgers.
- Listed the `openai/codex` tree at `90dd6cf9` (2026-11-19) and searched it for embedding code; read its model prompts. Read Aider's `repomap.py` at `5dc9490b` (2026-05-22).
- Read the papers' arXiv abstracts and result tables in their HTML versions, and the vendors' blog posts and docs on 2026-11-19. Quotes from the Cursor forum are staff posts. The Boris Cherny quote comes from a secondary report, because the original talk was not reachable.
- Converted token counts to local time with the lab speeds. Every such figure is marked as an estimate.

## Sources

Papers:

- Husain et al., CodeSearchNet Challenge, 2019: https://arxiv.org/abs/1909.09436
- Zhang et al., RepoCoder, EMNLP 2023: https://arxiv.org/abs/2303.12570
- Jimenez et al., SWE-bench, ICLR 2024: https://arxiv.org/abs/2310.06770
- Wang et al., CodeRAG-Bench, v2 2025-02-26: https://arxiv.org/abs/2406.14497
- Xia et al., Agentless, v2 2024-10-29: https://arxiv.org/abs/2407.01489
- Chen et al., LocAgent, v2 2025-04-29: https://arxiv.org/abs/2503.09089
- Peng et al., SWE-QA, v2 2026-04-26: https://arxiv.org/abs/2509.14635
- Zhang et al., CORE-Bench, v3 2026-08-24: https://arxiv.org/abs/2606.11864
- Qin and Xie, Agent Retrieval Bench, v1 2026-07-27: https://arxiv.org/abs/2607.24882
- Oskooei et al., Deep Agentic Search for Repository-Level Code Question Answering, v1 2026-08-02: https://arxiv.org/abs/2608.01507
- Xue et al., Embeddings vs. Grep: Rethinking the Dense Advantage in Code Search, v1 2026-11-16: https://arxiv.org/abs/2611.20455

Vendor posts and docs:

- Anthropic, How Claude Code works in large codebases, 2026-05-14: https://claude.com/blog/how-claude-code-works-in-large-codebases-best-practices-and-where-to-start
- Anthropic, Building agents with the Claude Agent SDK, 2025-09-29: https://claude.com/blog/building-agents-with-the-claude-agent-sdk
- OfficeChai, report on Boris Cherny's remarks, 2025-05-27 (secondary): https://officechai.com/ai/claude-researcher-explains-how-agentic-search-performed-better-than-rag-for-code-generation/
- Cursor, Improving agent with semantic search, 2025-11-06: https://cursor.com/blog/semsearch
- Cursor, Securely indexing large codebases, 2026-01-27: https://cursor.com/blog/secure-codebase-indexing
- Cursor, Fast regex search, 2026-03-23: https://cursor.com/blog/fast-regex-search
- Cursor, semantic search docs, read 2026-11-19: https://cursor.com/docs/context/semantic-search
- Cursor forum, staff posts on removing codebase indexing, 2026-07-21 to 2026-07-31: https://forum.cursor.com/t/what-do-you-think-about-cursor-removing-the-codebase-indexing-settings/165899
- Sourcegraph, How Cody understands your codebase, 2024-02-15: https://sourcegraph.com/blog/how-cody-understands-your-codebase
- Sourcegraph, Cody FAQ 5.7: https://5.7.sourcegraph.com/cody/faq
- Continue, deprecated `@Codebase`: https://docs.continue.dev/reference/deprecated-codebase
- Aider, repository map docs: https://aider.chat/docs/repomap.html
- Amp manual: https://ampcode.com/manual

Source code:

- Keiko at `bc57815c`: https://github.com/oscharko-dev/Keiko/tree/bc57815cbb383faa6531dd416afa5fe6ccd3d56a
- Codex CLI at `90dd6cf9`: https://github.com/openai/codex/tree/90dd6cf9df1d3236a7f8085f65eb76c75277646f
- Aider at `5dc9490b`: https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/repomap.py
- CodeSearchNet repository (archived 2023-04-11): https://github.com/github/CodeSearchNet

Earlier research in this repository: `docs/research/github-ingestion.md`, `docs/research/graph-knowledge.md`, `docs/research/memory-fit.md`, `docs/research/local-evaluation.md`, `docs/research/stack-scoring.md` and `docs/research/test-dataset.md` on their `research/*` branches.
