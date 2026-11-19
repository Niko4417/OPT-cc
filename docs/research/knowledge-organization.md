# How a project memory should organize its knowledge as it grows

Research for [issue #33](https://github.com/Niko4417/OPT-cc/issues/33), part of the map in [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). Sources checked on 2026-11-19. Keiko was read from `oscharko-dev/Keiko` branch `dev` at commit [`bc57815`](https://github.com/oscharko-dev/Keiko/tree/bc57815cbb383faa6531dd416afa5fe6ccd3d56a) (2026-11-19). Paths written as `Keiko:path` refer to that commit.

## Question

How should one project memory organize its knowledge so answer quality stays high as it grows? The ticket asks three things:

1. Does retrieval quality or latency fall as one index grows? This covers distractor and corpus-size studies, the 2025 LIMIT paper on single-vector embeddings, and approximate nearest neighbour (ANN) recall against index size.
2. Does routing questions and content to separate indexes help, for example one index per source type or per topic with a router per question, compared with one index plus metadata filters? Measured results count, vendor claims do not.
3. What are Keiko's Knowledge Pods today, next to Foundry IQ knowledge bases and Bedrock Knowledge Bases? Are the Keiko team's three claims right?

The ticket does not ask whether human-maintained Knowledge Pods are useful. [ADR-0011](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0011-one-app-to-ask-and-add-knowledge-and-notes-are-project-artifacts.md) already settles that one app is used to ask and to add project artifacts and notes. A sibling ticket ([#32](https://github.com/Niko4417/OPT-cc/issues/32)) covers code retrieval against recursive search. This note touches code only where index size or routing is the point.

## Answer

Keep one index per project memory, tag every chunk with metadata, and route at two points: by content type at ingestion, and softly by metadata at query time. Do not split a project memory into separately routed indexes for quality.

- Quality does fall as one index grows, for every retriever. More near-miss chunks compete for the same top-k slots. In one controlled test, growing a corpus from 54 to 1,128 documents cut recall of the answer-bearing document from 0.975 to 0.750 ([Subedi et al. 2026](https://arxiv.org/abs/2606.11350)). Dense retrieval loses ground to BM25 as the index grows ([Reimers and Gurevych 2021](https://arxiv.org/abs/2012.14210)). The measured fixes are a lexical lane next to the dense lane, a reranker and metadata filters.
- The LIMIT paper does not argue for smaller indexes. Its limit depends on how many combinations of relevant documents queries ask for. Real embedding models fail its task even on 46 documents, while BM25 reaches 97.8% recall@2 and a reranker solves it ([Weller et al. 2025](https://arxiv.org/abs/2508.21038)). The lesson is to keep BM25 and a reranker, not to split the index.
- Latency and ANN recall are not a problem at project memory size. The test dataset is about 5 million tokens, which is 10,000 to 50,000 chunks. Exact search fits that size. Keiko's own run measured HNSW recall@10 of 1.000 at 50,000 vectors with a 15 ms median ([Keiko ADR-0164](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/adr/ADR-0164-one-bounded-in-memory-usearch-hnsw-runtime.md)).
- No independent study shows that routing to separate indexes beats one combined search. Searching all medical corpora together beat every single corpus ([MedRAG](https://arxiv.org/abs/2402.13178)). Querying all sources beat a learned router by 1.0 to 1.3 points ([RAGRoute](https://arxiv.org/abs/2502.19280)). Hard routing with a weak router cost 21.8 points of correctness, while soft routing, which fuses a scoped list with a global list, gained 4.5 points ([Subedi et al. 2026](https://arxiv.org/abs/2606.11350)). Metadata filters chosen by a model inside one index raised Hits@10 from 0.74 to 0.90 ([Multi-Meta-RAG](https://arxiv.org/abs/2406.13213)).
- The only large routing gain comes from Microsoft: +36% answer relevance for Foundry IQ knowledge bases at medium effort over searching all sources ([Microsoft Foundry blog, 2025-11-18](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/-/4470720)). That gain mixes source selection with query planning and a second retrieval round, and Microsoft ran it on mostly internal datasets with an unnamed judge model. It is a claim for the bake-off to test, not evidence.

Verdicts on the Keiko team's claims:

| Claim | Verdict | Short reason |
|---|---|---|
| Knowledge Pods were not made for one app where a consultant both asks questions and adds knowledge | Partly confirmed | Keiko asks and manages pods in one desktop app, but a pod indexes local folders, repositories and file lists by path. A chat cannot add a document or typed note to a pod. Chat attachments expire after 30 minutes, and chat learning goes to a personal memory vault. |
| Extending pods for that would copy Foundry's functions | Partly confirmed | Keiko's contracts already copy Foundry IQ's object model, in their own words. The functions a pod lacks (file upload, managed connectors, model-based source selection, a semantic reranker) exist in Foundry IQ and in Bedrock. The argument holds only if the provider review picks a cloud. On-device is still open ([ADR-0008](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0008-leave-the-installation-location-open-until-the-provider-review-and-bake-off.md)). |
| Content needs routing before it is embedded so quality stays high | Partly confirmed for parsing, contradicted for separate indexes | Type-aware parsing and chunking before embedding helps: AST chunking adds 4.3 points recall@5 on code ([cAST](https://arxiv.org/abs/2506.15655)). Splitting content into separately routed indexes did not beat one combined index in any independent study found. |

## Does quality or latency fall as one index grows?

### Retrieval quality against corpus size

Every study found shows quality falling as distractors are added with queries held fixed. The size of the drop depends on the retriever.

| Study | Setup | Result |
|---|---|---|
| [Reimers and Gurevych, ACL 2021](https://arxiv.org/abs/2012.14210) | MS MARCO passages, index grown from 10,000 to 8.8 million, MRR@10 | BM25 falls from 79.9 to 17.6. A dense model without hard negatives falls from 89.0 to 17.3 and drops below BM25 at 8.8 million. A dense model trained with hard negatives falls from 91.5 to 28.6 and stays ahead. Lower dimensions give more false positives. |
| [Subedi et al., arXiv 2606.11350, v2 2026-10-14](https://arxiv.org/abs/2606.11350) | One state transport agency's document corpus, 32 queries, distractors added from 54 to 1,128 documents, BGE-M3, k = 10 | Precision@10 falls from 0.759 to 0.281 and recall from 0.975 to 0.750. The authors also apply a query-free dilution measure to seven agency corpora. |
| [Shao et al. (MassiveDS), arXiv 2407.12854, 2024](https://arxiv.org/abs/2407.12854) | Datastore grown to 1.4 trillion tokens of mixed sources | Downstream task quality rises with datastore size without clear saturation. More useful content outweighs more distractors when the extra content holds answers. |

The two results fit together. A larger index helps when the added content answers questions, and it hurts when the added content only competes. A project memory grows with both kinds: new decision records and project artifacts answer questions, while old issues, superseded decision records and boilerplate code compete.

Distractors also hurt after retrieval. The model reads a chunk that looks relevant but holds no answer and gets worse. [Cuconasu et al. (SIGIR 2024)](https://arxiv.org/abs/2401.14887) found such passages reduce accuracy, while adding random passages raised it by up to 35%. [Jin et al. (2024)](https://arxiv.org/abs/2410.05983) found answer quality first rises and then falls as more passages are retrieved, and named hard negatives as the main cause. Both effects grow with index size, because a larger index holds more near-misses.

### The LIMIT paper

[Weller et al., "On the theoretical limitations of embedding-based retrieval"](https://arxiv.org/abs/2508.21038) (v1 2025-08-28, v2 2026-03-12, ICLR 2026) proves that the embedding dimension bounds how many distinct top-k sets a single-vector model can return. The numbers:

- With vectors optimized freely on the test set and k = 2, the largest solvable corpus grows with the cube of the dimension. Extrapolated, that is about 4 million documents for 1,024 dimensions and 107 million for 3,072. These are extrapolations from dimensions 4 to 45.
- On the LIMIT dataset (50,000 documents, 1,000 queries of the form "who likes X", k = 2), the tested single-vector models reach 5 to 19% recall@100. BM25 reaches 93.6% and the multi-vector GTE-ModernColBERT 54.8%.
- On LIMIT small (46 documents, same queries), the single-vector models reach only 19 to 54% recall@2, BM25 97.8%. A Gemini-2.5-Pro reranker solved all 1,000 queries.

For a project memory, this means:

- A project memory of 10,000 to 100,000 chunks is far below the theoretical bound for common dimensions. Index size is not where the limit bites.
- The limit bites on queries that combine attributes, such as "decision records that mention both the mail store and Kotlin". Those fail even in tiny indexes. Splitting into smaller indexes does not fix them.
- The fixes the authors name are a lexical lane, multi-vector models and rerankers. Keiko already has the first and third.

### ANN recall and latency against index size

- HNSW search cost grows with the logarithm of the index size ([Malkov and Yashunin](https://arxiv.org/abs/1603.09320)).
- FAISS recommends a flat index, which is exact search, when exact results are needed or when few searches are made, and HNSW for small data sets with enough memory ([FAISS index guidelines](https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index)). Its size thresholds start at 1 million vectors.
- Keiko searches partitions of up to 20,000 vectors exactly and uses HNSW above that, then re-scores the HNSW candidates exactly (`Keiko:packages/keiko-local-knowledge/src/retrieval/usearch-ann-index.ts`, `DEFAULT_EXACT_SCAN_THRESHOLD = 20_000`). Its qualification run on 2026-07-27 with 50,000 vectors of 384 dimensions measured recall@10 of 1.000, an HNSW median of 15 ms against 54 ms for exact search, and a 39 s build (`Keiko:docs/adr/ADR-0164-one-bounded-in-memory-usearch-hnsw-runtime.md`, D6). This is the Keiko team's own measurement.

Size of one project memory. The test dataset in [docs/research/test-dataset.md](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md) has 12.4 MB of Kotlin and Java, 0.9 MB of Markdown and about 5.7 MB of issue and pull request text. That is about 5 million tokens, or 10,000 to 50,000 chunks at 100 to 500 tokens each. A client system ten times larger stays below 1 million chunks. At this size, ANN recall and search latency do not limit quality. Model calls take far longer than the search.

Filters change this picture. A metadata filter that keeps few chunks makes HNSW slow or inaccurate: post-filtering HNSW often fails to reach high recall, and its cost grows as the share of matching items falls. Pre-filtering, which scans only the matching items exactly, always has perfect recall and is cheap when few items match ([ACORN, SIGMOD 2024](https://arxiv.org/abs/2403.04871)). For a project memory, the safe choice is to filter first and scan the matching chunks exactly when the filter keeps fewer than about 20,000 chunks.

## Does routing to separate indexes help?

### Measured results

| Study | What was routed | Baseline | Result |
|---|---|---|---|
| [MedRAG, Xiong et al. 2024](https://arxiv.org/abs/2402.13178) | Questions to one of four medical corpora | All four corpora searched together (MedCorp) | The combination scored highest for every retriever: 71.6 average accuracy with RRF over four retrievers, against 69.7 for PubMed, the best single corpus, with the same retriever. The authors recommend the combination. |
| [RAGRoute, Dhasade et al., v2 2026-04-09](https://arxiv.org/abs/2502.19280) | Questions to 4 to 13 sources by a small neural classifier | Query all sources | Accuracy 65.6 against 67.0 on MIRAGE and 76.1 against 77.1 on MMLU. Routing saved 20 to 81% of traffic and lost 1.0 to 1.3 points. The paper frames routing as a cost measure. |
| [RouterRetriever, Lee et al., AAAI 2025](https://arxiv.org/abs/2409.02685) | Questions to one of several domain-specific embedding models | One multi-task embedding model | +3.2 nDCG@10 on BEIR. This routes between embedding models, not between corpora. |
| [Subedi et al. 2026](https://arxiv.org/abs/2606.11350) | Questions to a domain scope, applied as a metadata filter on one index | Unscoped search | Hard routing lowered correctness by 21.8 points on a nine-corpus benchmark, where a 7B model router picked the right scope only 33% of the time. Soft routing (scoped results fused with global results by RRF) never lowered correctness and gained 4.5 points there. On the agency's own corpus, no scoped variant beat unscoped search on correctness. |
| [Multi-Meta-RAG, Poliakov and Shvai 2024](https://arxiv.org/abs/2406.13213) | Questions to metadata filters (news source, date) extracted by an LLM, inside one index | Unfiltered search | Hits@10 from 0.749 to 0.891 with bge-large and from 0.742 to 0.904 with voyage-02. GPT-4 answer accuracy from 0.56 to 0.61. |
| [FeB4RAG, Wang et al. 2024](https://arxiv.org/abs/2402.11891) | Resource selection over 16 BEIR sub-collections | Naive federated search | A benchmark for federated search in RAG. It does not compare against one merged index. |

What these results show:

- A wrong route is expensive and a missed route cannot be recovered. Router accuracy or source-selection recall ranged from 33 to 90% in these studies. A project memory has few labelled questions to train a router on.
- Searching everything and letting ranking sort it out was as good as or better than hard routing in every independent comparison.
- Narrowing the search helps when the narrowing is soft or exact: a metadata filter the question clearly implies, or a scoped list fused with a global list.
- Separate embedding models per domain gave small gains (RouterRetriever). Those need separate vector spaces, which one index can hold as separate vector fields fused by rank. Azure AI Search allows up to 10 vector fields in one vector query ([service limits](https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity)), and Keiko already fuses embedding lanes by rank (`Keiko:docs/local-knowledge/knowledge-pods.md`, "Retrieval and score semantics").

### Vendor results and guidance

Microsoft's Foundry IQ team reports that medium reasoning effort beats minimal effort by 21 points on average (59 to 80, a 36% relative gain) across five datasets ([Microsoft Foundry blog, 2025-11-18](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/-/4470720)). Minimal effort runs the user's query against all knowledge sources. Medium effort adds LLM query planning, source selection and one follow-up retrieval. Limits of this evidence:

- It is the vendor's own evaluation. Three of the five datasets were built by the authors. The judge model is not named.
- It cannot separate source selection from query planning and the second retrieval round. Query decomposition alone helps multi-part questions, and Keiko has a deterministic version (`Keiko:packages/keiko-local-knowledge/src/retrieval/query-decomposition.ts`).
- The authors say gains concentrate on hard queries. They also report that a source's name matters most for selection, and that long source descriptions can hurt it.

The vendors' own documentation leans toward fewer indexes:

- Microsoft: use one indexed pipeline for content that shares a schema and management needs, and separate indexes for different schemas, processing or governance ([knowledge source overview](https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-overview), 2026-11-09). Its cost advice is to reduce the number of knowledge sources and consolidate content ([agentic retrieval overview](https://learn.microsoft.com/en-us/azure/search/agentic-retrieval-overview), 2026-10-29).
- AWS: one managed knowledge base holds up to 200 data sources in one managed datastore ([managed knowledge base quotas](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-quotas.html)). Routing across knowledge bases exists for agentic retrieval over up to 5 knowledge bases ([agentic retrieval](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-agentic-retrieve.html)).

### Routing at ingestion

"Routing before embedding" can mean two things. Sending each source to a parser and chunker that fits its type has evidence. Sending each type to its own index does not.

- Chunking code at syntax tree boundaries instead of fixed line windows added 4.3 points recall@5 on RepoEval and 2.67 points Pass@1 on SWE-bench ([cAST, 2025](https://arxiv.org/abs/2506.15655)).
- Code retrieval is hard for general retrieval models, according to the CoIR benchmark ([CoIR, ACL 2025](https://arxiv.org/abs/2407.02883)). A code-specific embedding model may help, as a second lane in the same index.
- Keiko already routes by type at ingestion. Its code parser cuts source files at declarations, and its document parsers keep pages, slides and table rows as citation locations (`Keiko:docs/local-knowledge/knowledge-pods.md`, "Project document formats"). All of it lands in the same pod store under one embedding model per pod.

## Keiko pods, Foundry IQ and Bedrock side by side

| | Keiko Knowledge Pods (`dev`, 2026-11-19) | Foundry IQ knowledge bases | Bedrock Knowledge Bases |
|---|---|---|---|
| Unit of content | A pod ("capsule" internally) with sources of kind `folder`, `repository` or `files`, each a local path (`Keiko:packages/keiko-contracts/src/local-knowledge.ts`). Connector pods sync Jira and Confluence. | A knowledge source: an indexed source backed by a search index (index, blob, Azure SQL, file upload, OneLake, SharePoint) or a remote source queried live (SharePoint remote, web, MCP server, Fabric, Work IQ). | A data source inside a knowledge base. Managed knowledge bases have 7 connectors (S3, SharePoint, Confluence, web crawler, Google Drive, OneDrive, custom). Customer-managed ones have S3 and custom. |
| Grouping | A pod set holds pod ids only, no copied vectors (`Keiko:docs/local-knowledge/knowledge-pod-sets-ledger.md`). | A knowledge base references up to 10 knowledge sources. Each indexed source has its own index. | A managed knowledge base holds up to 200 data sources in one datastore. Agentic retrieval spans up to 5 knowledge bases. |
| Storage and index | One encrypted SQLite store per state directory, partitioned by pod. Exact search up to 20,000 vectors per partition, in-memory HNSW above (`Keiko:docs/adr/ADR-0164-one-bounded-in-memory-usearch-hnsw-runtime.md`). | Azure AI Search index with keyword, vector and hybrid queries. | Managed: an auto-scaling datastore run by AWS. Customer-managed: OpenSearch Serverless, Aurora, Neptune Analytics and others. |
| Who picks what to search | The person attaches pods or pod sets to a chat. Pods carry `alwaysQuery` and `sourceRoutingInstructions` fields, but code search found no consumer outside composition and tests (`Keiko:packages/keiko-local-knowledge/src/composition.ts`). Every attached pod is searched. | `minimal` searches all sources. At `low` and `medium` an LLM selects sources from names, index descriptions and `retrievalInstructions`, by default up to 3 or 5 in the GA API 2026-10-01 ([limits](https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity)). `alwaysQuerySource` forces a source in. | A model plans sub-queries per knowledge base after a probe search across knowledge bases. Metadata filters are set by hand or generated by a model from a schema ([query configuration](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-config.html)). |
| Query planning | Deterministic split of chained questions into up to 3 parts, plus an optional model-based query transformer (`Keiko:packages/keiko-local-knowledge/src/retrieval/query-decomposition.ts`). | LLM planning into parallel sub-queries. `medium` adds one follow-up retrieval. | Managed: model decomposition, iterative retrieval and a sufficiency check. Customer-managed: optional query decomposition. |
| Ranking | Lexical and dense lanes fused by RRF, optional reranker through the gateway, optional entailment check per claim. | Semantic ranker on each sub-query, results merged. | Managed reranker by default in managed knowledge bases, optional reranker models otherwise. |
| Adding knowledge from the app | Point a pod at local files and index them. Chat attachments live in a separate store for at most 30 minutes (`Keiko:packages/keiko-server/src/conversation-attachment-store.ts`). Chat learning writes facts and preferences to a personal memory vault, not to pods (`Keiko:docs/memory/memoria-viva-learning-passes.md`). | File knowledge source: direct upload, 200 files per source in GA, 10,000 in the 2026-11-01 preview. Or push documents into your own index. | Direct ingestion API adds, updates and deletes documents in a knowledge base with a custom or S3 data source without a sync ([direct ingestion](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion.html)). |
| Location | Local only. Owner decision of 2026-11-04: no sharing, no remote or managed-service pods (`Keiko:docs/local-knowledge/knowledge-pods.md`, "Local-only scope"). | The client's or consultancy's Azure tenant. | The client's or consultancy's AWS account. |

Three observations follow from the table.

- Keiko modelled pods on Foundry IQ on purpose. The contract file says KnowledgeSource, KnowledgeCapsule and CapsuleSet follow "Foundry IQ composition", and that one global knowledge pool is "unrepresentable in the type system" (`Keiko:packages/keiko-contracts/src/local-knowledge.ts`). The source routing validator cites Foundry IQ's "no global pool" rule (`Keiko:packages/keiko-local-knowledge/src/source-routing-validation.ts`).
- The "no global pool" rule pushes toward many small indexes and routing between them. The evidence above does not support that for quality inside one project memory. The project memory itself is the right boundary, because the model policy and the project memory bundle both work per project memory.
- Foundry IQ and Bedrock both offer the add path that ADR-0011 needs: upload a file or push a document into the index the questions run against. Keiko has no such path into a pod today.

## Keiko team claims in detail

### "Pods were not made for one app where a consultant both asks questions and adds knowledge"

Partly confirmed.

- Supported. A pod's sources are local paths, and deleting a pod leaves the files on disk. A chat cannot add a document or a typed note to a pod. Chat attachments are transient, with a hard 30-minute ceiling. MemoriaViva learns from chat into a personal vault in later passes, separate from pods. Pods are local only, with no sharing roadmap.
- Not supported. Keiko is one desktop app in which the same person creates pods, adds folders and asks questions. Pod management and chat are separate screens of one app, not separate apps.
- What ADR-0011 needs and Keiko lacks: an upload or note action that writes into the index questions run against, and storage of the uploaded copy instead of a path to the consultant's disk.

### "Extending pods for that would copy Foundry's functions"

Partly confirmed.

- Keiko's pods already copy Foundry IQ's object model, by Keiko's own comments in the contracts.
- The functions an extension would add exist in Foundry IQ (file knowledge source, indexers, semantic ranker, source selection) and also in Bedrock (direct ingestion, connectors, reranker, agentic retrieval).
- The claim matters only if the provider review picks Microsoft. ADR-0009 also names AWS and OVHcloud, and ADR-0008 keeps on-device open. On-device, no managed service exists to copy.
- The extension needed for ADR-0011 is small: an ingestion path from the app into the index. The routing functions are not needed, according to the evidence above, so copying them is optional.

### "Content needs routing before it is embedded so quality stays high"

Partly confirmed for type-aware parsing, contradicted for separate indexes.

- Type-aware parsing and chunking before embedding has measured gains ([cAST](https://arxiv.org/abs/2506.15655)). Keiko already does it.
- A domain-specific embedding model gave +3.2 nDCG@10 in one study ([RouterRetriever](https://arxiv.org/abs/2409.02685)). That needs a second vector lane, not a second index.
- Splitting content into separately routed indexes did not beat one combined search in any independent study found ([MedRAG](https://arxiv.org/abs/2402.13178), [RAGRoute](https://arxiv.org/abs/2502.19280), [Subedi et al.](https://arxiv.org/abs/2606.11350)). Hard routing with an inaccurate router lost 21.8 points.
- Quality does fall as an index grows ([Subedi et al.](https://arxiv.org/abs/2606.11350), [Reimers and Gurevych](https://arxiv.org/abs/2012.14210)). The measured remedies are lexical search next to dense search, rerankers, metadata filters and soft scoping.
- Evidence gap: no study measured routing on a corpus that mixes code, documents, issues and transcripts of one software project. The bake-off has to measure it.

## Recommendation

1. One index per project memory. Each chunk carries metadata: source kind (code, document, decision record, workshop result, issue, pull request, project artifact, note), origin (repository or project artifact), path or URL, date, language and decision status. Separate indexes only for reasons other than quality: a different model policy, different access rights or a provider limit.
2. Route content by type at ingestion, into the same index. Code goes to a syntax-aware chunker, Markdown to a heading chunker, transcripts to a speaker-turn chunker, issues and pull requests with their comments. A code-specific embedding model enters only if the bake-off shows a gain, as a second vector field fused by rank.
3. Hybrid lexical and dense retrieval with RRF and a reranker in every arm. Keep the lexical lane even if dense wins on average, because of LIMIT-type questions and dilution.
4. Soft routing at query time. A planner may add a metadata filter that the question implies, such as source kind "decision record" or decision status "accepted". The filtered list is always fused with an unfiltered list. Filtered search scans the matching chunks exactly when fewer than about 20,000 remain.
5. Superseded decision records stay in the index. Their decision status is metadata, and the answer step marks them, as in [ADR-0003](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0003-settle-supersession-from-explicit-links-and-send-conflicts-to-a-human.md) and [ADR-0005](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0005-show-every-decision-in-a-decision-register.md).

For the bake-off ([issue #13](https://github.com/Niko4417/OPT-cc/issues/13)):

- Add a dilution test. Score the gold question set on the test dataset alone, then again after adding a distractor corpus of similar domain and 5 to 10 times the size, such as another Android mail client's repository. Report recall@10 and answer correctness for both runs. This measures growth directly instead of trusting papers on other corpora.
- If the provider review picks Microsoft, run the Foundry IQ arm at `minimal` and at `medium` on the same gold set. If it picks AWS, run Bedrock single-shot retrieval and agentic retrieval. Score per question type. This tests the vendor's +36% on project memory data.
- Run the cloud arm on one knowledge source or knowledge base and on a split per source kind. This is the only direct test of the Keiko team's routing claim.

## Open points

- No independent study measured index routing on a mixed software project corpus. The verdict on separate indexes rests on medical, news and public-agency corpora.
- The Foundry IQ +36% and the Keiko HNSW numbers are measured by their makers. Neither was reproduced for this note.
- Microsoft's evaluation does not isolate source selection from query planning. A `low` effort run with one knowledge source would separate the two.
- GitHub code search may miss files. The finding that `alwaysQuery` and `sourceRoutingInstructions` have no runtime consumer in Keiko rests on code search on 2026-11-19 plus reading `composition.ts` and `retrieval-runner.ts`.

## Method

- Read the ticket, [GLOSSARY.md](https://github.com/Niko4417/OPT-cc/blob/main/GLOSSARY.md), the ADRs on `main` and the stack scoring research on `research/stack-scoring`.
- Read Keiko through the GitHub API at commit `bc57815` on `dev`: the pod documentation, contracts, composition, retrieval and ANN code, ADR-0036, ADR-0164, the chat attachment store and the MemoriaViva documentation. Used GitHub code search for `alwaysQuery`, `sourceRoutingInstructions`, `sourceRoutingByCapsule`, `alwaysQueryCapsuleIds` and `Foundry`.
- Read the arXiv abstract and HTML pages of each paper for its numbers. The Reimers and Gurevych tables come from the ar5iv rendering of the paper.
- Read Microsoft Learn and AWS documentation pages on 2026-11-19. Their dates come from the page metadata where shown.
- Ran no experiments. All numbers are as published.

## Sources

Papers and preprints:

- Weller, Boratko, Naim, Lee. On the theoretical limitations of embedding-based retrieval. v1 2025-08-28, v2 2026-03-12, ICLR 2026: https://arxiv.org/abs/2508.21038
- Reimers, Gurevych. The curse of dense low-dimensional information retrieval for large index sizes. ACL 2021: https://arxiv.org/abs/2012.14210 (tables via https://ar5iv.labs.arxiv.org/html/2012.14210)
- Subedi, Abdelaty, Sheshappanavar. When more documents hurt RAG: measuring and reducing vector search dilution with domain-scoped retrieval. v2 2026-10-14: https://arxiv.org/abs/2606.11350
- Shao et al. Scaling retrieval-based language models with a trillion-token datastore. 2024-07-09: https://arxiv.org/abs/2407.12854
- Cuconasu et al. The power of noise: redefining retrieval for RAG systems. SIGIR 2024: https://arxiv.org/abs/2401.14887
- Jin, Yoon, Han, Arik. Long-context LLMs meet RAG. 2024-10-08: https://arxiv.org/abs/2410.05983
- Malkov, Yashunin. Efficient and robust approximate nearest neighbor search using HNSW graphs. v4 2018-08-14: https://arxiv.org/abs/1603.09320
- Patel, Kraft, Guestrin, Zaharia. ACORN: performant and predicate-agnostic search over vector embeddings and structured data. SIGMOD 2024: https://arxiv.org/abs/2403.04871
- Simhadri et al. Results of the Big ANN: NeurIPS'23 competition. 2024-09-25: https://arxiv.org/abs/2409.17424
- Xiong, Jin, Lu, Zhang. Benchmarking retrieval-augmented generation for medicine (MedRAG). 2024-02-23: https://arxiv.org/abs/2402.13178
- Dhasade et al. Efficient federated search for retrieval-augmented generation using lightweight routing (RAGRoute). v2 2026-04-09: https://arxiv.org/abs/2502.19280
- Lee, Soldaini, Cohan, Seo, Lo. RouterRetriever: routing over a mixture of expert embedding models. AAAI 2025: https://arxiv.org/abs/2409.02685
- Wang, Khramtsova, Zhuang, Zuccon. FeB4RAG: evaluating federated search in the context of RAG. 2024-02-19: https://arxiv.org/abs/2402.11891
- Poliakov, Shvai. Multi-Meta-RAG: improving RAG for multi-hop queries using database filtering with LLM-extracted metadata. v2 2024-08-19: https://arxiv.org/abs/2406.13213
- Zhang et al. cAST: enhancing code retrieval-augmented generation with structural chunking via abstract syntax tree. v2 2025-10-02: https://arxiv.org/abs/2506.15655
- Li et al. CoIR: a comprehensive benchmark for code information retrieval models. ACL 2025: https://arxiv.org/abs/2407.02883

Vendor documentation and claims (vendor sources, not independent):

- Microsoft Learn, agentic retrieval overview, 2026-10-29: https://learn.microsoft.com/en-us/azure/search/agentic-retrieval-overview
- Microsoft Learn, what is a knowledge source, 2026-11-09: https://learn.microsoft.com/en-us/azure/search/agentic-knowledge-source-overview
- Microsoft Learn, service limits including agentic retrieval limits, 2026-11-06: https://learn.microsoft.com/en-us/azure/search/search-limits-quotas-capacity
- Microsoft Foundry blog, Foundry IQ: boost response relevance by 36% with agentic retrieval, 2025-11-18 (updated 2025-12-02): https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/-/4470720
- AWS, Amazon Bedrock Knowledge Bases overview, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html
- AWS, build a managed knowledge base, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html
- AWS, service quotas for managed knowledge bases, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-quotas.html
- AWS, use agentic retrieval to query a knowledge base, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-agentic-retrieve.html
- AWS, configure and customize queries and response generation, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-config.html
- AWS, ingest changes directly into a knowledge base, read 2026-11-19: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion.html
- FAISS wiki, guidelines to choose an index, read 2026-11-19: https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index

Keiko (`oscharko-dev/Keiko`, `dev` at [`bc57815`](https://github.com/oscharko-dev/Keiko/tree/bc57815cbb383faa6531dd416afa5fe6ccd3d56a), 2026-11-19):

- `docs/local-knowledge/knowledge-pods.md`, `docs/local-knowledge/knowledge-pod-sets-ledger.md`, `docs/local-knowledge/knowledge-pod-retrieval-goldset-ledger.md`
- `docs/adr/ADR-0036-hybrid-grounding-reciprocal-rank-fusion.md`, `docs/adr/ADR-0164-one-bounded-in-memory-usearch-hnsw-runtime.md`
- `docs/memory/memoria-viva-learning-passes.md`
- `packages/keiko-contracts/src/local-knowledge.ts`, `packages/keiko-contracts/src/local-knowledge-pods.ts`
- `packages/keiko-local-knowledge/src/composition.ts`, `source-routing-validation.ts`, `knowledge-pods.ts`
- `packages/keiko-local-knowledge/src/retrieval/scoped-vector-search.ts`, `usearch-ann-index.ts`, `vector-index.ts`, `retrieval-runner.ts`, `query-decomposition.ts`
- `packages/keiko-server/src/conversation-attachment-store.ts`

Earlier research in this repository: `docs/research/stack-scoring.md`, `docs/research/test-dataset.md` and `docs/research/graph-knowledge.md` on their `research/*` branches.
