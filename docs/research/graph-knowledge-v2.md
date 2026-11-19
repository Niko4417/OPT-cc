# Where graphs and graph databases fit with a cloud provider

Research for [issue #35](https://github.com/Niko4417/OPT-cc/issues/35), part of the map in [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). It extends the earlier [graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md) for [issue #15](https://github.com/Niko4417/OPT-cc/issues/15), here called "the earlier research", and does not repeat its evidence. Sources checked on 2026-11-19. Prices come from the providers' machine-readable price lists on that date, unless a row says otherwise.

## Question

The installation may now run on one cloud provider ([ADR-0008](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0008-leave-the-installation-location-open-until-the-provider-review-and-bake-off.md), [ADR-0009](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0009-choose-one-cloud-provider-by-research-and-measure-its-managed-retrieval.md)). The candidates are Microsoft, AWS and OVHcloud. With that in mind:

- Where does a graph help the project memory: code, decision records, issues and pull requests, workshop results, or across them?
- What is Graphify, how does it build its graph, under which license, and with what measured results?
- Does the project memory need a graph database, or do the link graph and the static code graph fit in plain tables? If a graph database helps, which one fits each cloud candidate?
- What exactly are Bedrock Knowledge Bases GraphRAG and the graph feature Microsoft presented for Foundry?
- Does the earlier finding hold, that LLM-built graphs are not worth 25 to 40 hours of local indexing, when a cloud model does the indexing of `thunderbird/thunderbird-android`?

## Answer

A cloud model removes the time and cost objection to LLM-built graphs. It does not remove the quality objection, so the earlier design stands: a link graph and a static code graph in plain tables, no graph database, and an LLM-built graph only as a measured arm in the bake-off.

- Indexing the whole test repository (about 5.2 million tokens) with LightRAG costs about €3.70 to €5.60 on OVHcloud AI Endpoints, $17 with GPT-5.6 Luna on Azure in the EU data zone, and $7.70 with Claude Haiku 5.5 or $77 with Claude Haiku 4.5 on Bedrock in Frankfurt. Microsoft GraphRAG costs about 1.4 times as much. Quotas set the time, not hardware: about 7 hours on a new Azure subscription with GPT-5.6 Luna (100,000 tokens per minute), 22 minutes with GPT-4.1 mini, and roughly 40 minutes on OVHcloud. The earlier research estimated 25 hours for the same job on the Mac. All figures are estimates from published token ratios and price lists, not measurements (section "Indexing re-check with cloud models").
- The quality evidence has not changed. LLM-built graphs still lose to plain retrieval on single-fact and unanswerable questions and win a few points on multi-hop questions in independent studies (earlier research, section "Evidence by question type"). Neither managed graph feature below has an independent benchmark.
- Graphify ([Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify), Apache-2.0, v0.9.112 of 2026-11-17) builds its code graph with tree-sitter and no language model, and sends documents to a language model for a "semantic pass". Its published results come from its own harness, and its code result rests on 6 questions. One independent blog test found no advantage over careful search. It is a second candidate for the static code graph, next to codebase-memory-mcp from the earlier research.
- The project memory does not need a graph database. The link graph has about 8,000 nodes for the test repository, and a code graph of its 3,796 source files has tens of thousands of nodes. One- and two-hop traversals over that size run as recursive SQL queries in SQLite or Postgres. If a graph database is ever wanted, Postgres with Apache AGE fits Microsoft, Neptune fits AWS, and OVHcloud offers no managed graph engine.
- Microsoft's feature is "knowledge graphs in Foundry IQ", in public preview since Ignite on 2026-11-17. It has two parts. A topic graph samples evidence across themes. An entity graph lets a language model extract entities and relations from Blob Storage content against an ontology you write in JSON-LD, and you query it in natural language or GQL. The feature itself costs nothing during preview, but you pay for the model tokens.
- Bedrock Knowledge Bases GraphRAG (generally available since 2025-03-07) extracts entities with a model you choose (Claude 3 Haiku, Claude Haiku 4.5 or Amazon Nova) and stores the graph and the vectors in Neptune Analytics. It reads only from S3, takes at most 1,000 files per data source unless AWS raises the limit, and offers no settings for the graph build. The smallest Neptune Analytics graph costs $0.581 an hour in Frankfurt, about $424 a month.

## What changed against the earlier research

| Earlier finding | With a cloud provider |
|---|---|
| LightRAG on the full repo takes about 25 hours of local generation, Microsoft GraphRAG about 40 | 22 minutes to 10 hours, set by the provider's rate quota. Money cost is €4 to $116 depending on provider, model and method |
| Index only Markdown, issues and pull requests for LightRAG in the bake-off, to save hours | The full repo is affordable. Code chunks still need a custom entity profile, and a static code graph still answers dependency questions better |
| Do not add Microsoft GraphRAG, partly because of 40 hours and unreliable JSON from local models | The time argument is gone. The quality argument from the earlier research still applies, and managed graph features now cover the same ground on two candidates |
| No graph database is named; codebase-memory-mcp uses SQLite | Confirmed: no graph database is needed (section "Graph databases") |
| A managed graph layer was not in scope | Foundry IQ knowledge graphs and Bedrock GraphRAG can join the bake-off as configurations of the provider's managed retrieval arm, as ADR-0009 allows |
| Kùzu was archived on 2025-10-10 | Still archived on 2026-11-19 (`gh api repos/kuzudb/kuzu`) |

## Where a graph helps in this architecture

| Source | Graph that helps | Built by | Questions it answers |
|---|---|---|---|
| Code | Static code graph: files, classes, functions, calls, imports, inheritance | A parser (tree-sitter, SCIP, Joern), no language model | "What calls or depends on X", "what breaks if `legacy/storage` goes" |
| Decision records | Link graph from front matter: `status`, `date`, `supersedes` ([ADR-0001](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0001-read-decision-records-in-common-formats-and-write-madr.md), [ADR-0003](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0003-settle-supersession-from-explicit-links-and-send-conflicts-to-a-human.md)) | The ingestion job | "Which decision is current", the decision register and decision conflicts |
| Issues and pull requests | Link graph from references (`#1234`, ADR and RFC numbers) and the files each pull request changed | The ingestion job, from the GitHub API | "Which issues and pull requests implement this decision", "which code did it touch" |
| Workshop results | The same link graph, when the Markdown format of [ADR-0002](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0002-prescribe-a-markdown-format-for-workshop-results.md) carries references | The ingestion job | "Which workshop discussed this decision" |
| Across sources | The link graph joins decision records, issues, pull requests, files and workshop results. The static code graph hangs below the file nodes | Both of the above | "Decision to issue to pull request to code", one or two hops |
| Prose with implicit links | An LLM-built entity graph (LightRAG, Foundry IQ entity graph, Bedrock GraphRAG) | A language model | Multi-hop questions that no explicit reference covers, such as a workshop statement that contradicts a decision without naming it |

Only the last row needs a language model. The decision register and the supersession rules of ADR-0003 depend on explicit links, and an extractor can invent a link that is not there. An LLM-built graph can add candidates for a human to confirm, the way ADR-0004 treats decision logs, but it should not decide supersession.

## Graphify

The ticket's "Graphify" is [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify). The older address `safishamsi/graphify` redirects to it: the GitHub API returns the same repository, created 2026-04-03. The README names Graphify-Labs as "the official source repository" and the PyPI package as `graphifyy`. Facts below come from the README, `BENCHMARKS.md`, `ARCHITECTURE.md` and source files at the default branch on 2026-11-19.

What it does. A user types `/graphify .` in Claude Code, Codex, Cursor, Gemini CLI or about 20 other assistants. Graphify writes `graphify-out/graph.json`, `graph.html` and `GRAPH_REPORT.md`, and the assistant then runs `graphify query`, `graphify path A B` and `graphify explain X` against the graph instead of reading files. An optional hook in Claude Code steers search tool calls toward the graph. An MCP server (`serve.py`) and exports to Neo4j and FalkorDB are optional extras.

How it builds the graph. The pipeline in `ARCHITECTURE.md` is `detect()` to `extract()` to `build()` to `cluster()` to the report, with NetworkX graphs between the stages:

- Code: tree-sitter parsing for about 40 languages, Java and Kotlin included (`_KOTLIN_CONFIG` and `_import_kotlin` in [`graphify/extract.py`](https://github.com/Graphify-Labs/graphify/blob/HEAD/graphify/extract.py)). Edges are `calls`, `imports`, `inherits` and similar. Each edge carries a tag: `EXTRACTED` when the source states it, `INFERRED` when Graphify's resolver derived it. No language model runs on code. The README says "nothing leaves your machine" for code.
- Documents, PDFs and images: a language model extracts nodes, edges and up to 3 hyperedges per chunk, with the prompt in [`graphify/llm.py`](https://github.com/Graphify-Labs/graphify/blob/HEAD/graphify/llm.py). Backends include Azure OpenAI, Bedrock, Ollama and OpenAI-compatible endpoints, which covers OVHcloud AI Endpoints. The default model for the Azure backend is `gpt-4o`.
- Communities: Leiden clustering if the `leiden` extra is installed, otherwise Louvain from NetworkX, with labels that need no language model.

License. The `LICENSE` file is Apache-2.0. `NOTICE` says code contributed before a relicensing stays available under MIT, kept in `LICENSE-MIT`. Graphify Labs also sells Graphify Cloud, which the README advertises in several places.

Measured results, all from Graphify's own harness in [`BENCHMARKS.md`](https://github.com/Graphify-Labs/graphify/blob/HEAD/BENCHMARKS.md), last updated 2026-07-05:

- LOCOMO (300 questions): 45.3% QA accuracy against 43.3% for hybrid retrieval with reciprocal rank fusion, 41.3% for dense retrieval and 49.7% for supermemory. These are conversation-memory questions, not code questions.
- LongMemEval-S (50 questions): 76%, tied with dense retrieval.
- Code: on ERPNext, a coding agent with one Graphify tool covered 82.0% of key facts against 70.8% for grep and read, on 6 questions.
- Kimi K2.6 is the answer model and the judge in the memory benchmarks. In the code benchmark, Claude Opus 4.8 is the agent, limited to 14 turns. The authors report 90.6% agreement between their judge and a second judge.
- The memory benchmarks use a BGE-m3 embedder and hybrid retrieval, while the README says "No embeddings, no vector store". The two documents describe different configurations.

Independent results are thin. A DEV Community post by Amit (2026-07-25) tested one 299-file repository. Graphify built 4,005 nodes and 10,229 edges in 7.1 seconds. On one question it cut context from 15,186 tokens (four full files) to 1,618, but used 9% more tokens than "disciplined search" (1,488). Median task time was 17.5 s for the baseline and 16.5 s with Graphify on four tasks. In a second run the baseline covered 3 of 3 answers fully and the Graphify wrapper 2 of 3. The author concludes Graphify "does not yet show" an advantage over careful search ([post](https://dev.to/amitrix/does-graphify-actually-help-an-ai-coding-agent-4bpi)). This is a blog with a small sample, not a study.

Fit for the project memory:

- Graphify's code pass is a static code graph, the kind the earlier research recommends. It is a candidate next to codebase-memory-mcp. Neither has an independent evaluation on Kotlin.
- Its document pass is an LLM-built graph and falls under the model policy. Without `--backend`, `graphify extract` picks the first backend whose API key is set, in the order Gemini, Kimi, Claude, OpenAI, DeepSeek, Azure, Bedrock and Ollama, and the README warns that Kimi routes to servers in China. Pin `GRAPHIFY_BACKEND` to the chosen provider, or run `--code-only`.
- Graphify gives coding agents a graph of a code folder. It has a pull request dashboard (`graphify prs`), but I found no path that loads GitHub issues or pull requests into the graph, and it has no decision status or supersession model. It cannot replace the link graph. A coding agent that uses it on a project memory still falls under [ADR-0007](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0007-coding-agents-access-a-project-memory-only-within-its-model-policy.md).

## Graph databases

Size of the graphs for the test repository, from the [test dataset research](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md):

- Link graph: 3,796 code files, 128 Markdown files, 22 decision records and about 4,000 issues and pull requests since 2024. That is about 8,000 nodes. The edges are references and changed files, likely tens of thousands.
- Code graph: Graphify produced 22,620 nodes and 48,710 edges for 3,758 ERPNext files in its own benchmark, and 4,005 nodes for 299 files in the blog test. For 3,796 Kotlin and Java files, expect 20,000 to 50,000 nodes. This is an extrapolation.

A table of nodes and a table of edges with indexes on both ends answers a one- or two-hop expansion with one recursive SQL query. codebase-memory-mcp already stores its code graph in SQLite. Nothing in the questions of the earlier research needs graph algorithms such as PageRank or community detection at query time. A graph database adds a second store to back up, export into the project memory bundle and delete from, for no question it alone can answer.

If a graph database is ever wanted, here is what each candidate offers:

| Store | Microsoft | AWS | OVHcloud | License and status |
|---|---|---|---|---|
| Apache AGE (openCypher in Postgres) | Generally available on Azure Database for PostgreSQL flexible server since 2025-05-19 ([announcement](https://techcommunity.microsoft.com/blog/adforpostgresql/general-availability-of-graph-database-support-in-azure-database-for-postgresql/4413894), [docs](https://learn.microsoft.com/en-us/azure/postgresql/azure-ai/generative-ai-age-overview)) | Not in the RDS for PostgreSQL extension tables for versions 16 to 19 ([list](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html)) | Not in the Public Cloud Databases extension list, updated 2026-11-19 ([list](https://docs.ovhcloud.com/en/guides/public-cloud/databases/postgresql-extensions)) | Apache-2.0, active |
| Neptune Database and Neptune Analytics | None | Managed. Analytics from 16 m-NCU at $0.581 an hour in Frankfurt, a paused graph at 10% ([pricing](https://aws.amazon.com/neptune/pricing/), AWS Price List) | None | Proprietary |
| Cosmos DB for Apache Gremlin | Managed. Its Learn pages now point to Graph in Microsoft Fabric for OLAP and for migrating Gremlin applications ([overview](https://learn.microsoft.com/en-us/azure/cosmos-db/gremlin/overview)) | None | None | Proprietary |
| Neo4j | AuraDB on Azure | AuraDB on AWS | Self-hosted on instances or Managed Kubernetes | AuraDB Professional from $0.09 per GB-hour ([pricing](https://neo4j.com/pricing/)); Community Edition is GPL-3.0 |
| FalkorDB | FalkorDB Cloud or self-hosted | FalkorDB Cloud or self-hosted | Self-hosted | SSPL (earlier research) |
| Kùzu | None | None | None | Archived 2025-10-10 |

OVHcloud's managed database engines are Cassandra, ClickHouse, Kafka, M3DB, MongoDB, MySQL, OpenSearch, PostgreSQL, Redis and Valkey (OVHcloud price catalog API). None is a graph engine. Its PostgreSQL offers `ltree` and `pgrouting`, which do not provide openCypher. Plain tables in its managed PostgreSQL cover the link graph and the code graph.

Microsoft also has Graph in Microsoft Fabric with GQL. Secondary sources disagree on whether it is generally available and whether GQL is still in preview, and I did not verify it against a primary source. It belongs to Fabric, not to Foundry IQ, and the project memory has no other reason to use Fabric.

## Managed graph features

### Foundry IQ knowledge graphs (Microsoft)

What Microsoft presented. The Foundry blog post "Foundry IQ: Introducing knowledge graphs, engineered for agents" of 2026-11-17 announced knowledge graphs and a retrieval-planning model, Microsoft Retriever 1, both in public preview ([post](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/foundry-iq-introducing-knowledge-graphs-engineered-for-agents/4572683)). The Ignite 2026 Book of News says Foundry IQ "now provides an enterprise knowledge graph out of the box" and lists knowledge graphs as preview ([Book of News](https://news.microsoft.com/events/ignite/2026/book-of-news/)). Microsoft Learn now titles its Azure AI Search pages "Foundry IQ (Azure AI Search)", and documents the feature there under REST API version `2026-11-01-preview`.

It has two separate graphs ([overview](https://learn.microsoft.com/en-us/azure/search/knowledge-graph-features-overview), updated 2026-11-07):

- A topic graph groups content into communities in a topic hierarchy. Retrieval samples evidence across topics for broad questions. It needs no ontology. A topic index with centroid vectors must first be prepared with the Foundry IQ skills plugin, and during preview a request must set `topicGraph.kind` to `diverse` itself, because the knowledge base does not detect broad questions ([how-to](https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-topic-graph)). The docs do not say whether a language model builds the topics. The blog says topics come "from your existing embeddings".
- An entity graph holds typed entities and relations ([how-to](https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-entity-graph), updated 2026-11-17). A chat model extracts them during indexing against an ontology you supply as constrained JSON-LD with OWL classes and properties. The examples use `gpt-5.6-luna` for extraction and `text-embedding-3-small` for vectors. A knowledge base translates natural-language questions into GQL, or the application sends GQL itself. Each result row carries provenance down to the source chunk and blob URL.

Limits of the entity graph in this preview, from the how-to:

- Content must come from an Azure Blob Storage knowledge source. ADLS Gen2, ACL ingestion and sensitivity-label ingestion are not supported. Blob soft delete must be on and blob versioning off.
- It needs a new knowledge base. You cannot enable it on an existing one.
- It is API-only, apart from a UI in the Foundry IQ skills plugin.
- The ontology must be fixed at creation unless you configure a reprocessing cache at creation. You cannot add that cache later or remove an attached ontology.
- "During preview, there's no additional charge for the entity graph feature itself. You still pay for the model tokens used." Microsoft gives no token ratio and asks users to estimate cost from corpus size and the number of extracted entities.
- No benchmark is published. An InfoWorld report notes the same for Retriever 1.

Fit. An ontology for the project memory could declare decision records, issues, pull requests, modules and the relations between them. But for those types the ingestion job already reads the relations from front matter and the GitHub API, exactly and without tokens. The entity graph helps only where the relation sits in prose. Issues and pull requests would first have to be exported as files into Blob Storage. It is a cheap configuration to try in the bake-off, not a replacement for the link graph.

### Bedrock Knowledge Bases GraphRAG (AWS)

AWS announced the preview in December 2024 ([what's new](https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-bedrock-knowledge-bases-graphrag-preview)) and general availability on 2025-03-07 ([what's new](https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-graphrag-generally-available/)). From the user guide ([overview](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html), [create](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs-build.html)):

- During ingestion a foundation model extracts "entities, facts, and relationships" from each chunk, with the setting `"method": "CHUNK_ENTITY_EXTRACTION"`. The allowed models are Claude 3 Haiku, now Legacy, Claude Haiku 4.5 and the Amazon Nova family. Claude Haiku 5.5 is not in the list.
- Neptune Analytics stores both the graph and the vectors. At query time the knowledge base runs a vector search, then follows graph edges from the hit chunks to related chunks and adds them to the context.
- Some models run only through cross-Region inference profiles. The data stays in the source Region, but a Global profile can send inference to any commercial Region. That matters for the model policy.
- GraphRAG is available in Frankfurt, London, Ireland, Oregon, N. Virginia, Tokyo and Singapore.

Limits, quoted from the user guide:

- "Configuration options to customize the graph build are not supported."
- "Autoscaling is not supported for Amazon Neptune Analytics graphs."
- "GraphRAG only supports Amazon S3 as the data source."
- "Each data source can have up to 1000 files." AWS can raise this to 10,000, or the bucket can be split into folders. The test repository has about 7,900 files once issues and pull requests are exported, so it needs 8 folders or a limit increase.
- With hierarchical chunking, GraphRAG returns only child chunks.
- Deleting the knowledge base does not delete the Neptune Analytics graph, which keeps billing.

Pricing. AWS charges nothing extra for GraphRAG beyond the underlying services. Those are the extraction model tokens, the embedding model ($0.20 per million tokens for Titan Text Embeddings V2 in Frankfurt, about $1 for the whole repo) and the Neptune Analytics graph. A 16 m-NCU graph, the API minimum, costs $0.581 an hour in Frankfurt, which is $424 for a 730-hour month. A paused graph costs $0.0581 an hour but cannot answer queries. I could not find out which capacity "Quick create" picks.

Evidence. AWS publishes no accuracy numbers for the managed feature. A Classmethod blog post (2026-07-20) built both a vector and a GraphRAG knowledge base over 66 synthetic documents with Claude Haiku 4.5 as the graph model. It reports no scores, only three questions. Both knowledge bases answered the single-hop question. Vector retrieval failed both multi-hop questions. GraphRAG answered them only after the author added a reranker. The author saw entity extraction break the graph into fragments where IDs were not linked ([post](https://dev.classmethod.jp/articles/graphrag-intro-03-bedrock-kb-graphrag-vs-vector-rag/)). An AWS blog post by the partner Lettria reports gains for Lettria's own hybrid GraphRAG, not for the managed feature; I could not load its body to check the numbers ([post](https://aws.amazon.com/blogs/machine-learning/improving-retrieval-augmented-generation-accuracy-with-graphrag/)). This is weak evidence.

### OVHcloud

OVHcloud has no managed retrieval layer and no managed graph feature. The price catalog lists no graph engine. A graph on OVHcloud means LightRAG, HippoRAG 2 or Graphify running on OVHcloud compute and calling AI Endpoints, with the graph in its managed PostgreSQL.

## Indexing re-check with cloud models

### Inputs

- Corpus and token ratios: the same as the earlier research. That is 5.2 million tokens for the full repo, 1.65 million without code, and 0.23 million for the Markdown alone. Per corpus token, LightRAG uses 7.2 input and 1.26 output tokens, HippoRAG 2 uses 5.1 and 0.83, and Microsoft GraphRAG uses 9.6 and 2.12 (GraphRAG-Bench, [arXiv 2506.05690](https://arxiv.org/abs/2506.05690), Table 10, measured with GPT-4o-mini).
- Prices per million tokens, input and output, for EU deployments:

| Provider | Model | Input | Output | Source |
|---|---|---|---|---|
| Microsoft | GPT-5.6 Luna, Data Zone Standard | $0.22 | $1.32 | Azure retail prices API, meters "5.6 luna ShortCo Inp/Opt Std DZ", Sweden Central, effective 2026-08-01 |
| Microsoft | GPT-4.1 mini, Data Zone Standard | $0.44 | $1.76 | Azure retail prices API, effective 2025-04-01 |
| Microsoft | GPT-4.1 mini, Data Zone Batch | $0.22 | $0.88 | Azure retail prices API |
| AWS | Claude Haiku 4.5, regional, Frankfurt | $1.10 | $5.50 | AWS Price List, `AmazonBedrockFoundationModels`, eu-central-1, 2026-11-19 |
| AWS | Claude Haiku 5.5, regional, Frankfurt | $0.11 | $0.55 | Same |
| AWS | Amazon Nova 2 Lite, Frankfurt | $0.429 | $3.597 | AWS Price List, `AmazonBedrock`, eu-central-1 |
| AWS | Amazon Nova Lite, Frankfurt | $0.078 | $0.312 | Same |
| OVHcloud | gpt-oss-120b | €0.08 | €0.40 | OVHcloud price catalog API, subsidiary DE, excluding VAT |
| OVHcloud | Qwen3-Coder-30B-A3B-Instruct | €0.06 | €0.22 | Same |
| OVHcloud | Mistral-Small-3.2-24B-Instruct | €0.09 | €0.28 | Same |
| OVHcloud | Llama-3.3-70B-Instruct | €0.67 | €0.67 | Same |

The OVHcloud catalog gives prices in units of 10⁻⁸ euro per token, and I converted them. The resulting €0.08 and €0.40 for gpt-oss-120b match the figures third-party price trackers show. OpenAI's own page lists GPT-5.6 Luna at $0.20 and $1.20, the same as Azure Global Standard.

### Money

Estimated model cost of one full indexing run, before retries:

| Scope | Method | Tokens in / out | Azure GPT-5.6 Luna | Azure GPT-4.1 mini batch | Bedrock Haiku 4.5 | Bedrock Haiku 5.5 | Bedrock Nova Lite | OVHcloud gpt-oss-120b | OVHcloud Qwen3-Coder-30B |
|---|---|---|---|---|---|---|---|---|---|
| Full repo, 5.2M | GraphRAG | 49.9M / 11.0M | $25.53 | $20.68 | $115.54 | $11.55 | $7.33 | €8.40 | €5.42 |
| Full repo, 5.2M | LightRAG | 37.4M / 6.6M | $16.89 | $14.00 | $77.22 | $7.72 | $4.96 | €5.62 | €3.69 |
| Full repo, 5.2M | HippoRAG 2 | 26.5M / 4.3M | $11.53 | $9.63 | $52.91 | $5.29 | $3.42 | €3.85 | €2.54 |
| Docs, issues and PRs, 1.65M | LightRAG | 11.9M / 2.1M | $5.36 | $4.44 | $24.50 | $2.45 | $1.58 | €1.78 | €1.17 |
| Markdown only, 0.23M | LightRAG | 1.7M / 0.3M | $0.75 | $0.62 | $3.42 | $0.34 | $0.22 | €0.25 | €0.16 |

Managed options on the same basis, model tokens only:

- Bedrock GraphRAG: AWS does not publish how many tokens `CHUNK_ENTITY_EXTRACTION` uses. If it lies between HippoRAG 2 (one extraction call per chunk) and LightRAG, the full repo costs $53 to $77 with Haiku 4.5, $27 to $40 with Nova 2 Lite and $3.40 to $5 with Nova Lite. Add about $1 of embeddings and $424 a month for the smallest running Neptune Analytics graph. The graph, not the indexing, dominates the cost.
- Foundry IQ entity graph with GPT-5.6 Luna: on the same assumption, $11.50 to $17 for the full repo. The Azure AI Search service bills separately: Basic costs $0.101 an hour and S1 $0.336 an hour in Sweden Central, about $74 and $245 a month. The how-to does not state a minimum tier for the entity graph.

### Time

In the cloud, indexing time depends on the rate quota and on how many calls run in parallel.

- Azure quotas grow with tiers 1 to 6, assigned per subscription ([quotas](https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits), updated 2026-08-20). GPT-5.6 Luna in Data Zone Standard allows 100,000 tokens per minute in Tier 1 and 333,000 in Tier 2. LightRAG's 44 million tokens on the full repo then take 7.3 hours in Tier 1 and 2.2 hours in Tier 2. Microsoft GraphRAG takes 10.2 and 3.1 hours. GPT-4.1 mini in Data Zone Standard allows 2 million tokens per minute in Tier 1, which gives 22 minutes for LightRAG. The Foundry IQ entity graph uses the same deployments and the same quotas.
- Bedrock: the default cross-Region quota for Claude Haiku 4.5 is 5 million tokens per minute per Region ([quotas](https://docs.aws.amazon.com/general/latest/gr/bedrock.html)), about 9 minutes of tokens for LightRAG. The regional on-demand quota is per account and I did not find a default. AWS publishes no ingestion throughput for GraphRAG, so the managed run time is unknown.
- OVHcloud allows 400 requests per minute per model and project, with no token limit stated ([capabilities](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-capabilities)). LightRAG makes about 9,400 extraction calls on the full repo (earlier research), so the floor is 24 minutes. With 32 parallel calls of about 8 seconds each, it takes about 40 minutes. The 8 seconds is my assumption, not a measurement.
- LightRAG runs 4 model calls in parallel by default (`MAX_ASYNC_LLM=4` in [env.example](https://github.com/HKUDS/LightRAG/blob/main/env.example)). It must be raised for cloud runs.

### Caveats

- The token ratios come from GPT-4o-mini on a novel. Code chunks may hold more entities per token, which raises output tokens, and output is 40 to 55% of the cost in the LightRAG rows above.
- GPT-5.6 Luna, gpt-oss-120b and Claude Haiku 5.5 are reasoning models, and reasoning tokens bill as output. GPT-5.6 Luna accepts reasoning effort `none` ([model page](https://developers.openai.com/api/docs/models/gpt-5.6-luna)). gpt-oss-120b has no off setting, so its cost will be higher than the table shows.
- Each provider counts tokens with its own tokenizer, and the 5.2 million figure is a character-based estimate.
- Extraction quality depends on the model (earlier research, section "Multi-hop questions"). The cheapest rows use the smallest models. A run with Nova Lite or Qwen3-Coder-30B may cost little and still build a poor graph.
- Every change to the repository costs a partial re-index. LightRAG re-extracts changed documents at the same per-token price.

Conclusion of the re-check. The earlier research ruled out LLM-built graphs on two grounds, quality and 25 to 40 hours of local indexing. In the cloud, the second ground shrinks to tens of dollars and under an hour, given the right quota. The first ground still stands.

## Recommendation

1. Keep the link graph and the static code graph from the earlier research, stored as plain tables in the database the installation already uses: SQLite on-device, or the provider's managed PostgreSQL in the cloud. Do not add a graph database.
2. Test Graphify's code pass (`--code-only`) next to codebase-memory-mcp on the Kotlin and Java code of the test repository, for the dependency questions in the gold set. Run its document pass only with `GRAPHIFY_BACKEND` pinned to a model location within the model policy.
3. In the bake-off, if the review picks Microsoft, run the Foundry IQ arm twice: plain, and with an entity graph built from a small project ontology (decision record, issue, pull request, module, decision and the relations between them). If it picks AWS, run the Bedrock arm twice: with a vector store, and with GraphRAG on Neptune Analytics. In both cases, delete the graph after the bake-off unless it wins. Score per question type, as the earlier research requires.
4. Keep LightRAG as the self-run LLM-graph arm, now indexed on the full repo with the chosen provider's model. It costs under $20 on Microsoft and under €6 on OVHcloud. On OVHcloud it is the only LLM-graph arm, because OVHcloud has no managed graph feature.
5. Treat a cloud LLM graph as content sent to a third-party cloud model. It is allowed only under a model policy that permits that model location, like every other cloud model call.

## Open points

- No independent benchmark exists for Foundry IQ knowledge graphs, Bedrock GraphRAG or Graphify on code. The bake-off will be the first measurement for this use case.
- The token use of the two managed graph builds is unknown. A run on the Markdown subset (0.23 million tokens) would measure it for under $1 per provider.
- The capacity that Bedrock's "Quick create" gives a Neptune Analytics graph, and the minimum Azure AI Search tier for the entity graph, are not documented in the pages I read.
- Quota tiers depend on the subscription's history. A new Azure subscription may start in Tier 1 and need hours for a full GPT-5.6 Luna run.

## Method

I read issue #35, `GLOSSARY.md`, ADR-0001 to ADR-0011 on `origin/main` and the earlier research on `origin/research/graph-knowledge`. I identified Graphify through a web search, then read its repository metadata, README, `ARCHITECTURE.md`, `BENCHMARKS.md`, `LICENSE-MIT`, `NOTICE`, `graphify/llm.py` and `graphify/extract.py` through the GitHub API. For Microsoft, I read the Foundry blog post, the Ignite 2026 Book of News and three Microsoft Learn pages on knowledge graphs, plus the Azure OpenAI quota page. For AWS, I read two Bedrock user guide pages on GraphRAG and the Bedrock quota page. I took prices from the Azure retail prices API, the AWS Price List bulk files for eu-central-1 and the OVHcloud public cloud price catalog API, all on 2026-11-19, and computed costs with the token ratios of the earlier research. I looked for independent evaluations of each managed feature and of Graphify through web searches and found only the two blog posts cited. I did not run any indexing job.

## Sources

Microsoft:

- Foundry blog, "Foundry IQ: Introducing knowledge graphs, engineered for agents", 2026-11-17: https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/foundry-iq-introducing-knowledge-graphs-engineered-for-agents/4572683
- Microsoft Ignite 2026 Book of News: https://news.microsoft.com/events/ignite/2026/book-of-news/
- Knowledge graph features overview, updated 2026-11-07: https://learn.microsoft.com/en-us/azure/search/knowledge-graph-features-overview
- Create and query an entity graph, updated 2026-11-17: https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-entity-graph
- Configure topic graph retrieval, updated 2026-11-07: https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-how-to-topic-graph
- Azure OpenAI quotas and limits, updated 2026-08-20: https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits
- Azure retail prices API: https://prices.azure.com/api/retail/prices
- Apache AGE on Azure Database for PostgreSQL: https://learn.microsoft.com/en-us/azure/postgresql/azure-ai/generative-ai-age-overview and the GA announcement of 2025-05-19: https://techcommunity.microsoft.com/blog/adforpostgresql/general-availability-of-graph-database-support-in-azure-database-for-postgresql/4413894
- Cosmos DB for Apache Gremlin overview: https://learn.microsoft.com/en-us/azure/cosmos-db/gremlin/overview
- InfoWorld on Foundry IQ knowledge graphs and Retriever 1 (press): https://www.infoworld.com/article/4250726/microsoft-adds-knowledge-graphs-retrieval-model-to-foundry-iq-for-better-ai-grounding.html

AWS:

- Build a knowledge base with Neptune Analytics graphs: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html
- Create a knowledge base with Neptune Analytics graphs: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs-build.html
- GraphRAG preview announcement, December 2024: https://aws.amazon.com/about-aws/whats-new/2024/12/amazon-bedrock-knowledge-bases-graphrag-preview
- GraphRAG general availability announcement, 2025-03-07: https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-bedrock-knowledge-bases-graphrag-generally-available/
- Lettria on the AWS Machine Learning blog (partner claim): https://aws.amazon.com/blogs/machine-learning/improving-retrieval-augmented-generation-accuracy-with-graphrag/
- Bedrock endpoints and quotas: https://docs.aws.amazon.com/general/latest/gr/bedrock.html
- Neptune pricing: https://aws.amazon.com/neptune/pricing/
- AWS Price List bulk API, offers `AmazonBedrock`, `AmazonBedrockFoundationModels` and `AmazonNeptune`, eu-central-1: https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/index.json
- RDS for PostgreSQL extension versions: https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-extensions.html

OVHcloud:

- Public cloud price catalog API, subsidiary DE: https://eu.api.ovh.com/1.0/order/catalog/public/cloud?ovhSubsidiary=DE
- AI Endpoints capabilities and limits: https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-capabilities
- Public Cloud Databases for PostgreSQL extensions, updated 2026-11-19: https://docs.ovhcloud.com/en/guides/public-cloud/databases/postgresql-extensions

Models:

- OpenAI, GPT-5.6 Luna model page: https://developers.openai.com/api/docs/models/gpt-5.6-luna

Graphify and other tools:

- Graphify repository, v0.9.112 of 2026-11-17: https://github.com/Graphify-Labs/graphify (README.md, ARCHITECTURE.md, BENCHMARKS.md, NOTICE, LICENSE-MIT, graphify/llm.py, graphify/extract.py)
- LightRAG `env.example`: https://github.com/HKUDS/LightRAG/blob/main/env.example
- Apache AGE: https://github.com/apache/age
- Kùzu (archived): https://github.com/kuzudb/kuzu
- Neo4j pricing: https://neo4j.com/pricing/

Independent posts (weak evidence):

- Amit, "Does Graphify actually help an AI coding agent?", DEV Community, 2026-07-25: https://dev.to/amitrix/does-graphify-actually-help-an-ai-coding-agent-4bpi
- Classmethod, "GraphRAG 入門 #3", 2026-07-20: https://dev.classmethod.jp/articles/graphrag-intro-03-bedrock-kb-graphrag-vs-vector-rag/

Papers:

- Xiang et al., When to use Graphs in RAG (GraphRAG-Bench): https://arxiv.org/abs/2506.05690

Earlier research in this repository: `docs/research/graph-knowledge.md` on `research/graph-knowledge` and `docs/research/test-dataset.md` on `research/test-dataset`.
