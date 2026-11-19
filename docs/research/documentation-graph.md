# Whether a graph helps with domain knowledge and other documentation

Research for [issue #49](https://github.com/Niko4417/OPT-cc/issues/49), part of the map in [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). It extends the [graph knowledge research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge/docs/research/graph-knowledge.md) for issue #15, here called "the first graph research", and the [cloud graph research](https://github.com/Niko4417/OPT-cc/blob/research/graph-knowledge-v2/docs/research/graph-knowledge-v2.md) for issue #35, here called "the second graph research". It does not repeat their evidence. Sources checked on 2026-11-20. Repository licenses and versions come from the GitHub REST API on that date.

## Question

The owner's hypothesis from 2026-11-20 is that a graph may cost too much for ordinary code but may pay off for the knowledge work a project documents. Does a graph improve answers over plain hybrid retrieval for the documentation side of a project memory, and if so which graph, built how, and for which question types?

Documentation here means four kinds:

- Domain knowledge, the glossary's term for what the client's business does and how: business rules, processes, domain terms and functional specifications (German: Fachlichkeit).
- Decision records in large numbers, with supersession.
- General Markdown documentation with links between files.
- Agent instruction files in a repository: `AGENTS.md`, `CLAUDE.md`, skills and agent definitions.

## Answer

The hypothesis holds for one of the four kinds, in a form that needs no language model. For the other three the evidence contradicts it or is too thin to decide.

| Kind of documentation | Does a graph pay off? | Evidence |
|---|---|---|
| Decision records with supersession | Yes, the link graph from front matter and parsed references that the first graph research already recommends. No LLM-built graph is needed | One study by the method's own authors on versioned documents, one independent benchmark on outdated text. No study uses decision records or large numbers of them |
| Domain knowledge | Undecided. LLM-built graphs gain or lose 1 to 3 points on domain prose in independent tests. A term graph from a glossary has never been measured against anything | The evidence is thin. The largest gains come from company authors on private data |
| General Markdown documentation | No for link traversal. The measured gain is zero to small, and it disappears when the added tokens are counted. The heading path as text on each chunk has a small measured gain and needs no graph | Three small studies, two of them by the method's own authors |
| Agent instruction files | No. No tool retrieves them through a graph, and no study measures a gain for question answering. A source type value and a path scope on each chunk are enough | No measurement exists. The studies that exist are about coding task success, staleness and prompt injection |

The premise about code also needs a correction. Both earlier research files found that cost depends on how a graph is built, not on whether the content is code or documentation. A static code graph from a parser costs no model calls. An LLM-built graph over the 0.23 million tokens of Markdown in the test dataset costs $0.34 to $0.42 on Bedrock in Frankfurt (section "Cost and time for the test dataset"). Cost is therefore no reason to prefer a graph for documentation and no reason to avoid one. Quality decides.

Recommendations for the review to accept, change or reject:

1. Keep the design of the two earlier research files: a link graph and a static code graph in plain tables, no graph database, and an LLM-built graph only as a measured arm.
2. Extend the link graph to all Markdown, at no model cost: one node per file and per heading, edges for relative links and heading anchors, and the link's anchor text stored with the edge. For the 128 Markdown files of the test dataset a regular-expression pass found 213 file-to-file edges in 0.04 seconds.
3. Put the heading path and the document title into the text of every prose chunk. This is the one structure signal with a controlled positive result, and it works without any graph.
4. Measure a term graph in the bake-off as an add-on, built from a glossary a human confirmed. Each chunk of a decision record, document, workshop result or code file carries the ids of the glossary terms it mentions. No published result exists for this, so the bake-off would be the first measurement.
5. Give agent instruction files their own `source_type` value and a path scope, and no graph. Label their chunks in the answer prompt as quoted file content, because they are written as commands to an agent.
6. Add domain knowledge documents and a glossary, in German and English, to the synthetic companion repository. `thunderbird/thunderbird-android` contains neither, so the test dataset cannot test the domain knowledge part of the hypothesis today.
7. Compare every graph variant against the baseline at the same number of context tokens. Two of the studies below show a gain that an equally large plain retrieval also reaches.

## What the earlier research already settles

- LLM-built graphs lose to plain retrieval on single-fact and unanswerable questions and gain 1 to 13 points on questions that span several documents (first graph research, section "Evidence by question type").
- A link graph from decision record front matter (`status`, `supersedes`, date) and parsed references answers "which decision is current" and "which code and issues does a decision affect" with no model call (first graph research, section "Recommendation").
- The link graph and the static code graph fit in plain tables, and one indexing run of LightRAG over the whole test dataset costs $7.72 with Claude Haiku 5.5 in Frankfurt (second graph research, sections "Graph databases" and "Money").
- No paper measures the gain of the link graph (first graph research, section "Open points"). That is still true on 2026-11-20.

## What the test dataset offers

I measured the Markdown of `thunderbird/thunderbird-android` at commit [`1daaefb`](https://github.com/thunderbird/thunderbird-android/commit/1daaefba6a2994496ec2e3a73888584ec7f00f54) of 2026-11-19 with a script that reads links, headings and backticked names by regular expression (section "Method"). These are my own counts, not published figures.

| Measure | Value |
|---|---|
| Markdown files, bytes | 128 files, 897,604 bytes, about 0.23 million tokens |
| Headings | 1,488 |
| Files with YAML front matter | 1 |
| Link targets in total | 652, of which 296 external |
| Relative links to another Markdown file | 254, of which 32 name a heading anchor |
| Distinct file-to-file edges | 213, of which 79 start at `docs/SUMMARY.md`, the table of contents |
| Files with no link in or out | 35 of 128 |
| Largest connected group of files | 88 |
| Links to issues and pull requests of the repository | 30 |
| Links to code paths | 18 |
| Relative links whose target is missing | 19 |
| Time to read and parse all 128 files | 0.04 seconds on one core of the Mac |

For the 19 decision records (10 ADRs and 9 RFCs, templates left out):

- 18 link to at least one issue or pull request.
- 12 name at least one path, Gradle module or class in backticks that resolves to a file or folder in the tree. The median is 2 resolvable names per record.
- Of 37 class names that match a Kotlin or Java file name, 1 matches more than one file.
- 7 records name nothing that resolves to code.

For domain knowledge and agent instruction files:

- The repository has no glossary file. Its documentation is engineering documentation for an email client: architecture, contribution guides, release steps. Nothing in it is domain knowledge in the glossary's sense.
- The repository has one agent instruction file, a root [`AGENTS.md`](https://github.com/thunderbird/thunderbird-android/blob/1daaefba6a2994496ec2e3a73888584ec7f00f54/AGENTS.md) of 8,168 bytes and 27 headings. It links to two other Markdown files, and 9 of its 40 backticked names resolve to paths in the tree. No other Markdown file links to it. There is no `CLAUDE.md`, no skill, no agent definition, no Cursor rule and no Copilot instruction file.
- A substring count shows how term edges spread. "account" occurs in 53 of the 128 files, "legacy" in 42, "feature flag" in 17, "repository pattern" in 8, "global database" in 7 and "UUIDv7" in 4. I picked these six terms by hand as an illustration. A term graph would link 41% of all files through "account" and 3% through "UUIDv7".

Three things follow. Explicit links from decision records to issues, pull requests and code exist and can be parsed. The link graph between general documents is sparse, and a third of its edges come from one table of contents. Domain knowledge and instruction files are nearly absent, so questions about them need synthetic material.

## Evidence on domain and enterprise documentation

No benchmark covers business rules, process descriptions or functional specifications of a client system. The closest corpora are clinical guidelines, technical specifications, regulations, textbooks, support tickets and migration notes.

### LLM-built graphs on domain prose

| Study | Who | Corpus | Result |
|---|---|---|---|
| GraphRAG-Bench, medical set ([Xiang et al.](https://arxiv.org/abs/2506.05690), Table 2) | Independent benchmark of other groups' methods | NCCN clinical guidelines, English, corpus size not stated | Accuracy on fact retrieval, complex reasoning and contextual summarization: plain RAG with reranking 64.73, 58.64, 65.75. HippoRAG 2 66.28, 61.98, 63.08. LightRAG 63.32, 61.32, 63.14. Microsoft GraphRAG 38.63, 47.04, 41.87 |
| GraphRAG-Bench, textbooks ([Xiao et al.](https://arxiv.org/abs/2506.02404), Table 4) | Independent benchmark | 20 computer science textbooks, 1,018 questions. The only plain baselines are TF-IDF and BM25 | Average accuracy: BM25 71.66, TF-IDF 71.71, LightRAG 71.22, Microsoft GraphRAG 72.50, HippoRAG 72.64, RAPTOR 73.58. GPT-4o-mini without retrieval scores 70.68 |
| O-RAN specifications ([Ahmad et al.](https://arxiv.org/abs/2507.03608), Table II) | Independent of the tools tested | 74 specification documents, 600 multiple-choice questions | Factual correctness: vector 0.48, graph 0.50, hybrid 0.58, with standard deviations of 0.10 to 0.22 |
| Regulatory text, ObliQA ([Karkout et al.](https://aclanthology.org/2026.kallm-1.12/), Table 3) | Authors of a hybrid method, with GraphRAG and dense retrieval as baselines | 40 regulatory documents, tested only on the 188 questions the dense baseline failed | Overall score on a 1 to 5 scale: dense RAG 2.40, GraphRAG 2.41. GraphRAG took 311 seconds per question |
| Earnings call transcripts ([Sarmah et al.](https://arxiv.org/abs/2408.04948), Table 5) | Company authors (BlackRock, NVIDIA) | 50 transcripts, English | Answer relevancy: vector 0.91, graph 0.89, hybrid 0.96. Context recall: vector 1, graph 0.85 |
| ABAP migration notes ([Min et al.](https://arxiv.org/abs/2507.03226), Tables 1 and 2) | Company authors (SAP), private data | 550 PDF documents, about 2,000 chunks, 150 questions | Weighted semantic alignment: dense retrieval 50.80, graph by GPT-4o 65.83, graph by dependency parser 61.87. Faithfulness: 77.18, 74.24, 72.76 |

Reading the table:

- On the medical set, the best LLM-built graph gains 1.55 points on fact retrieval and 3.34 on complex reasoning and loses 2.67 on summarization against plain RAG with reranking. This matches the range the first graph research found on novels and Wikipedia. Domain text does not widen the gap.
- GraphRAG-Bench reports that the medical graph is sparser than the graph over novels: 11.8 entities per 1,000 tokens and an average degree of 1.05, against 19.6 and 2.27 ([Xiang et al.](https://arxiv.org/abs/2506.05690), Table 9). Structured domain text gives an extractor fewer relations to find, not more.
- The SAP result is the only large gain. Its baseline is one dense embedding model without reranking or lexical search, its questions and data are private, and the graph lowers faithfulness. It is company evidence, not independent evidence.

### Graphs from explicit structure

The largest reported gains on enterprise documents come from graphs that follow structure the documents already state. Every one of these results comes from the method's own authors.

- LinkedIn turned Jira tickets into trees of their sections and joined tickets by the tracker's explicit links and by title similarity. Rules read the predefined fields and a language model parsed the rest. Retrieval MRR rose from 0.522 to 0.927 against chunk retrieval with the same embedding model, and median resolution time fell by 28.6% in six months of use ([Xu et al.](https://arxiv.org/abs/2404.17723), Tables 1 and 3). The paper states neither the corpus size nor the test set size.
- Document GraphRAG links chunks by the chapter hierarchy and by shared keywords from the YAKE extractor, with no language model in the build. On 17 German production planning documents of over 5,500 pages, context recall rose from 0.729 to 0.794 and the correctness score from 0.505 to 0.571 at 10 retrieved chunks ([Knollmeyer et al.](https://www.mdpi.com/2079-9292/14/11/2102), Table 1). Plain retrieval with 20 chunks reaches a context recall of 0.829 in the same paper (Table 4), so part of the gain is more context. The graph with 20 chunks reaches 0.892.
- An agent over a graph built from structured fields of the Swiss commercial registry answered 60 questions with a correctness of 0.828 against 0.262 for hybrid retrieval ([Capozzi and Helbing](https://arxiv.org/abs/2605.18770), Table 7). The questions ask for entities and counts over registry records, which is closer to a database query than to document retrieval.

### German text

I found no published comparison of LightRAG, Microsoft GraphRAG, HippoRAG or another LLM-built graph against chunk retrieval on German documents. The search queries are listed in the section "Method". What exists:

- Document GraphRAG above. Its internal set is German, and its graph uses headings and keywords. The authors note that YAKE returned generic German terms such as "Allgemeine Anforderungen", and that keywords from a language model raised context recall from 0.794 to 0.809 (Table 5).
- A regex-built graph of 58 German federal statute codes with about 168,000 nodes and 22,000 citation edges ([Siddig et al.](https://arxiv.org/abs/2610.13695)). Sentence-level chunks cut along the statute structure beat 100-word chunks, 2.28 against 0.85 on the authors' token-normalized recall at paragraph depth. Following citation edges adds 0.12. The metric favors short chunks by construction, and the study has no lexical or hybrid baseline.
- LightRAG has a `SUMMARY_LANGUAGE` setting, default English, with German among the examples in [`env.example`](https://github.com/HKUDS/LightRAG/blob/main/env.example). Microsoft GraphRAG's prompt tuning has a `--language` option ([docs](https://microsoft.github.io/graphrag/prompt_tuning/auto_prompt_tuning/)). Neither project publishes a quality measurement for German.

### Vendor claims

These are often cited for "knowledge graphs help enterprise questions". None measures document retrieval for a project memory.

- data.world: GPT-4 answered 54.2% of 43 questions correctly through a knowledge graph view of an insurance database against 16.7% with SQL ([Sequeda et al.](https://arxiv.org/abs/2311.07509), Table 1). The task is querying structured data.
- Writer: 86.31 on RobustQA for its own retrieval against 75.89 for the next system ([Mozolevskyi and AlShikh](https://arxiv.org/abs/2405.02048), Table 2). The paper does not say how the graph is built or how answers were scored.
- Lettria on the AWS Machine Learning blog: 80% correct answers against 50.83% for a vector baseline ([post](https://aws.amazon.com/blogs/machine-learning/improving-retrieval-augmented-generation-accuracy-with-graphrag/)). This is unverified. The page body would not load, and the numbers come from search result snippets.

## Term graphs without a language model

A term graph links every chunk that mentions the same known term, across decision records, documents, workshop results and code. The terms come from a glossary. No paper measures glossary-based linking against LLM extraction for retrieval, and none measures it on German text. Three kinds of nearby evidence exist.

### Graphs from cheap extraction against LLM-built graphs

| Study | Graph built by | Result | Indexing cost |
|---|---|---|---|
| LinearRAG ([Zhuang et al.](https://arxiv.org/abs/2510.10114), Tables 1 and 2), own authors | spaCy named entities linked to the sentences and passages that contain them, no relations | GraphRAG-Bench medical: 63.72 against 61.68 for plain retrieval, 60.77 for HippoRAG 2 and 54.36 for LightRAG. 2Wiki: 63.70 against 43.00, 55.00 and 39.00 | 250 seconds and 0 model tokens on 2Wiki, against 1,147 seconds and 6.2 million tokens for HippoRAG 2 |
| KET-RAG, keyword part ([Huang et al.](https://arxiv.org/abs/2502.09304), Table 3), own authors | Every non-stop word linked to the chunks that contain it | HotpotQA F1 46.4 against 44.2 for text retrieval and 43.0 for Microsoft GraphRAG. MuSiQue F1 20.6 against 19.4 and 25.1 | $0.07 to $0.09 against $21 to $25 for Microsoft GraphRAG |
| SAP ([Min et al.](https://arxiv.org/abs/2507.03226), Table 2), company authors | spaCy dependency parser with rules | 61.87 against 65.83 for the GPT-4o graph, which is 94% | No model tokens for extraction. Entities still get embeddings |
| Code graphs ([Chinthareddy](https://arxiv.org/abs/2601.08773)), one author and one assessor | Tree-sitter | 43 of 45 questions correct against 38 for an LLM-built graph and 31 for vector retrieval, on three Java repositories | 2.81 seconds against 200.14 seconds on one repository |

In LinearRAG's run on the medical set, every LLM-built graph scores below plain retrieval, and only the graph without a language model scores above it. These results come from each method's own authors and from Wikipedia-style questions, apart from the medical set. They show that a graph without a language model can match an LLM-built graph. They do not show that either beats a tuned hybrid baseline on project documentation.

### How precise term matching is between prose and code

Software traceability research measures the task a term graph performs, which is linking documentation sentences to code. The numbers come from the Karlsruhe Institute of Technology group's own tools on five open-source Java projects, with baselines by other groups that the authors tuned per project.

| Approach | Precision | Recall | F1 |
|---|---|---|---|
| Vector space, LSI and similar retrieval between sentences and code files (TAROT) | 0.15 | 0.44 | 0.22 |
| Word embeddings (FTLR) | 0.15 | 0.43 | 0.21 |
| Fine-tuned CodeBERT | 0.24 | 0.45 | 0.28 |
| Name heuristics from prose straight to code (ArDoCode) | 0.27 | 0.78 | 0.37 |
| Prose to a list of named architecture elements, then names to code (TransArC) | 0.87 | 0.81 | 0.82 |
| Named architecture elements to code identifiers alone (ArCoTL) | 0.98 | 0.99 | 0.98 |

Source: [Keim et al., ICSE 2024](https://publikationen.bibliothek.kit.edu/1000165692), Tables 2 and 3, averages over the five projects. The follow-up with embeddings and GPT-4o mini reaches an average F1 of 0.217 for documentation to code on three of the projects and concludes that the result does not yet allow fully automatic use ([Fuchß et al., ICSE 2025](https://fuchss.org/assets/pdf/2025/icse-25.pdf), Table VI).

For a term graph this means that matching free prose against code by shared words is wrong about four times in five, while matching a short list of curated names against code identifiers is right 98% of the time. A glossary is such a curated list. This supports building the term graph from confirmed terms only and never from all nouns.

### Matching German terms

A German glossary term appears inflected ("Vertrag", "Verträge", "Vertrags") and inside compounds ("Vertragsverlängerung"). Exact string matching misses both.

| Tool | License | Latest release | What it gives |
|---|---|---|---|
| [explosion/spaCy](https://github.com/explosion/spaCy) | MIT. The German pipelines `de_core_news_*` 3.8.0 are MIT in their metadata | v3.8.18, 2026-10-26 | `PhraseMatcher` on lemmas. Self-reported German lemma accuracy 0.975 to 0.984 |
| [adbar/simplemma](https://github.com/adbar/simplemma) | MIT | v2.0.0, 2026-08-12 | Dictionary lemmatizer in pure Python. README: 0.97 on a German treebank. No compound splitting |
| [WojciechMula/pyahocorasick](https://github.com/WojciechMula/pyahocorasick) | BSD-3-Clause | v2.3.1, 2026-04-27 | Finds many terms in one pass. Matches inside words, so the caller must check word boundaries |
| [dtuggener/CharSplit](https://github.com/dtuggener/CharSplit) | MIT | none, last commit 2020 | German compound splitter, about 95% head detection by its README. Unmaintained |
| [wartaal/HanTa](https://github.com/wartaal/HanTa) | LGPL-3.0 | v1.2.0, 2026-04-26 | Lemma and morpheme segmentation. LGPL needs a license check before use |
| [LIAAD/yake](https://github.com/LIAAD/yake) | AGPL-3.0, with a paid commercial license | v0.7.3, 2026-02-09 | Keyword extraction as in Document GraphRAG. Fails the license rule |

The one independent comparison of German lemmatizers is from 2019 and covers older versions: 0.975 for RNNTagger and 0.891 for the spaCy of that year on 7,642 tokens of non-standard text ([Ortmann et al., KONVENS 2019](https://github.com/rubcompling/konvens2019)). No maintained compound splitter with a permissive license turned up. A glossary that lists each term's German and English labels and its known compound forms as aliases avoids the splitter. The repository's own `GLOSSARY.md` already lists avoided synonyms per term, which is the same idea.

## Links, headings and front matter between Markdown documents

No study measures Markdown relative links, heading anchors or front matter fields as graph edges. The nearest evidence:

- Link traversal on technical documentation. LARAG stores the outgoing links of each chunk of a Sphinx documentation site and expands the top hits along them, with no language model at indexing. On 20 expert questions it scores a BERTScore F1 of 0.8382 with 8 chunks, against 0.8378 for plain retrieval with 10 chunks and 0.8360 with 5 ([Bolognesi et al.](https://arxiv.org/abs/2605.07517), Table 7). Quality is equal within the third decimal. The saving is 18% of the context tokens. The authors include staff of the company whose documentation it is.
- Citation edges in German statutes. One hop along citation edges raises the authors' recall metric from 2.28 to 2.40. When the tokens of the added chunks are counted, every expansion strategy falls below its own baseline ([Siddig et al.](https://arxiv.org/abs/2610.13695)).
- Hyperlinks on Wikipedia. A trained retriever that walks Wikipedia hyperlinks reached 73.3 F1 on HotpotQA against 58.8 for the best earlier system, and author-written links beat links from an entity linker by 2.3 F1 ([Asai et al.](https://arxiv.org/abs/1911.10470), Tables 1 and 7). HotpotQA questions were written along those hyperlinks, and the retriever is trained, so this does not carry over to an untrained one-hop expansion.
- Anchor text on an intranet. On IBM's intranet of 4.6 million pages, the best ranking with anchor text had 24% to 87.5% higher recall at ranks 1 to 20 than the best ranking without it, for popular queries. PageRank added 0% to 13.6% ([Fagin et al., WWW 2003](https://mccurley.org/papers/www2003/p641-mccurley.html), Table 1). The useful part of a link was the words other pages use to describe the target, indexed as text, not the link structure. The study is from 2003, predates embeddings and is by IBM researchers on IBM's own system.
- Heading paths. A placebo-controlled study on 200 Wikipedia articles and 1,585 scientific papers found that chunks cut along sections, with the real heading path attached, beat fixed windows with context written by a language model by 0.022 and 0.012 nDCG@10. They beat the same chunks with shuffled heading paths by 0.010 and 0.016. A two-stage retrieval that first picks a section and then a chunk lost 0.033 and 0.015 ([Kuehlkamp et al.](https://arxiv.org/abs/2610.10170)). I read the abstract only. The effects are small and consistent.

Tools that build or walk such a graph without a language model:

| Tool | License | Status | Note |
|---|---|---|---|
| [executablebooks/markdown-it-py](https://github.com/executablebooks/markdown-it-py) with `mdit-py-plugins` | MIT | v4.3.0, 2026-11-10 | CommonMark parser with plugins for heading anchors and front matter |
| [run-llama/llama_index](https://github.com/run-llama/llama_index) `MarkdownNodeParser` | MIT | v0.14.26, 2026-11-13 | Splits on headings and writes the heading path as chunk metadata |
| [DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) | MIT | v0.12.1, 2026-10-16 | The static code graph candidate of the first graph research. Its README says it also indexes Markdown sections and adds a reference edge from a document to code when the target resolves to exactly one definition |
| [datastax/graph-rag](https://github.com/datastax/graph-rag) | Apache-2.0 | v0.8.0, 2025-04-04, no push since 2025-05-05 | Walks metadata edges in an existing vector store. The vendor publishes no answer quality number for it |
| [mfarragher/obsidiantools](https://github.com/mfarragher/obsidiantools) | BSD-3-Clause | 0.11.0, 2025-07-08 | Graph of wikilinks, backlinks, front matter and tags for an Obsidian vault |
| [basicmachines-co/basic-memory](https://github.com/basicmachines-co/basic-memory) | AGPL-3.0 | v1.0.0, 2026-10-29 | Fails the license rule |

The glue ingestion job of ADR-0018 already parses Markdown by heading. Reading links and anchors in the same pass needs a parser, not a framework.

## Decision records in large numbers

The first graph research covers the link graph for supersession. Two results published since add support, and one gap remains.

- A graph of explicit version structure beats both plain retrieval and an LLM-built graph on version questions. VersionRAG models documents, versions and changes as nodes and routes each question by intent. On 100 questions over 34 versioned technical documents with GPT-4o mini, it answers 90% of version listing questions correctly against 20% for plain RAG and 65% for a Neo4j LLM-built graph, and 65% of change questions against 35% for both. The LLM-built graph loses to plain RAG on ordinary content questions, 61.6% against 70.0%. VersionRAG's index took 186,000 tokens against 2.97 million for the LLM-built graph ([Huwiler et al.](https://arxiv.org/abs/2510.08109), Tables 4 and 5). The benchmark is small and built by the authors, and VersionRAG still uses a language model to read version metadata that decision records carry in front matter.
- Outdated text in the context measurably harms answers. On the HoH benchmark, adding one outdated passage next to the current one lowered the share of fully correct answers from 93.24% to 83.00% for Llama 3.1 70B and from 90.34% to 68.13% for Llama 3.1 8B ([Ouyang et al.](https://arxiv.org/abs/2503.04800), Tables 4 and 5). The corpus is Wikipedia. This is independent support for the rule of ADR-0003 that the prompt labels each decision record with its decision status and leads with the current one.
- No study measures retrieval over hundreds or thousands of decision records. The test dataset has 19 real records and a handful of synthetic ones, so it cannot test "large numbers" either. This part of the hypothesis stays open.

## Agent instruction files

### What a parser can read

Every format defines references that need no language model to read.

| Format | Machine-readable references | Source |
|---|---|---|
| `AGENTS.md` | Position in the directory tree only. The nearest file to the edited file applies. No front matter, no import syntax | [agents.md](https://agents.md) |
| Claude Code `CLAUDE.md` | `@path` imports up to four hops deep, files in subdirectories that load when a file there is read | [memory docs](https://code.claude.com/docs/en/memory) |
| Claude Code rules, skills, agent definitions | `paths` globs on rules. Skill front matter `name` and `description`, relative links to bundled files. Agent definition front matter with a `skills` list | [memory](https://code.claude.com/docs/en/memory), [skills](https://code.claude.com/docs/en/skills), [subagents](https://code.claude.com/docs/en/sub-agents) |
| Agent Skills specification | `name` must match the folder, `description` up to 1,024 characters, relative paths one level deep | [agentskills.io](https://agentskills.io/specification) |
| Codex | One `AGENTS.override.md` or `AGENTS.md` per directory from the project root down, joined in order, 32 KiB in total by default | [Codex docs](https://developers.openai.com/codex/guides/agents-md) |
| GitHub Copilot | `applyTo` globs in `.github/instructions/*.instructions.md` | [Copilot docs](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions) |
| Cursor | `globs`, `alwaysApply` and `description` in `.cursor/rules/*.mdc` | [Cursor docs](https://cursor.com/docs/context/rules) |

The common reference is a scope that says which paths a file applies to. Imports and links between instruction files are the second kind.

### What studies say about the files

All of the following are academic studies independent of the tool vendors, unless noted.

- Content. In 2,303 files from 1,925 repositories, 75.9% cover testing, 70.8% implementation details, 68.1% architecture, 65.1% the development process and 63.0% build and run commands. 14.8% cover security ([Chatlatanagulchai et al.](https://arxiv.org/abs/2511.12884), Table 3). The files are a compact source for "how do I build and test this" and "how is it structured".
- Overlap with other documentation. Context files written by a language model had no significant effect on task success (p = 0.87 and p = 0.37 on the two benchmarks) and raised cost by 20% to 23%. Files written by developers gave +2.4%, also not significant. When the authors removed all other documentation from the repositories, generated context files helped by 2.7% ([Gloaguen et al.](https://arxiv.org/abs/2602.11988), version 3). The authors read this as the files repeating what the documentation already says. Two authors work at a company that builds coding agents.
- References between files are rare. Across 4,768 context files in 2,586 repositories there are 497 references between context files, 301 of them from `CLAUDE.md` to `AGENTS.md`. Of 601 skills, 85.5% bundle no further files ([Galster et al.](https://arxiv.org/abs/2602.14690)). A graph of these references would have about one edge per five repositories.
- Staleness. 23.0% of 356 repositories had at least one stale reference in an instruction file, 230 of 18,048 checked references. A manual check of 50 flagged items confirmed 64% ([Treude and Baltes](https://arxiv.org/abs/2606.09090)).
- Agents obey a stale file over the code. In 320 runs on 40 synthetic projects, Claude Code and Codex CLI followed the stale instruction file in 90.1% of choices and the code in 4.2%. One added sentence that tells the agent to follow the code raised that share to 91.4% ([Xu and Wu](https://arxiv.org/abs/2610.30571)).

### How tools treat them

- Coding agents never search for these files. Every agent in the table above loads them by file name, tree position, glob match or import and puts them into the prompt. Skills are the exception. The model picks a skill by matching the task against the names and descriptions it holds in context ([agentskills.io](https://agentskills.io/specification)).
- Retrieval and graph tools index them as ordinary Markdown or not at all. LightRAG and Microsoft GraphRAG have no handling for them. Graphify writes its own section into `CLAUDE.md` and `AGENTS.md` and indexes the user's instruction files like any other Markdown file. A comment in its clustering code names the cost. A second pass has to split communities that hub nodes such as `CLAUDE.md` join because they connect to everything ([`graphify/cluster.py`](https://github.com/Graphify-Labs/graphify/blob/HEAD/graphify/cluster.py)).
- Search over skills exists for marketplaces, not repositories. SkillSeek indexes the name and description of 34,000 skills with BM25 and reaches a recall at 5 of 0.391, or 0.433 with a reranker. Adding the start of the skill body to the index lowered the agent's pass rate ([Yang et al.](https://arxiv.org/abs/2609.38822)).
- No paper, benchmark or tool report measures whether indexing instruction files as their own source, or boosting them, changes question answering quality.

### Risk

Instruction files are written as commands to an agent. A project memory that copies their text into an answer prompt hands those commands to its own answer model.

- A payload planted in a knowledge file that an instruction file merely references, without loading it, triggered unauthorized tool use in 100%, 90% and 80% of trials on three of four models ([Gadgil et al.](https://arxiv.org/abs/2607.14611), Table 1). This is the closest evidence to retrieved text. No study covers a retrieval system that quotes instruction files into an answer prompt.
- Hidden Unicode in rule files can carry instructions a reviewer does not see ([Pillar Security, 2025-03-18](https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents), vendor research).
- OWASP's guidance for indirect prompt injection is to separate and mark untrusted content ([LLM01:2025](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)).

### Finding

A graph over agent instruction files is not supported by any evidence. The files are few, their mutual references are rare, and the test dataset has one file with two links. Treating them as their own source type has no measured benefit either, but it costs one metadata value and gives three things a plain Markdown chunk lacks:

- A filter and a prompt label, so the answer model sees the text as a quoted repository file and not as its own instructions.
- A path scope per chunk, read from tree position or globs, so "which rules apply to `feature/account`" can be answered by a metadata match.
- A way to keep the app's own files out. The project memory folder that coding agents edit carries an `AGENTS.md` and a skill written by the app ([coding agents research](https://github.com/Niko4417/OPT-cc/blob/research/coding-agents/docs/research/coding-agents.md)). Those describe the app, not the modernization project, and should not be answered from.

## Cost and time for the test dataset

Sizes come from the first graph research: 0.23 million tokens of Markdown, 1.65 million tokens with issues and pull requests, 5.2 million with code. Token ratios per corpus token come from GraphRAG-Bench Table 10: LightRAG 7.2 input and 1.26 output, HippoRAG 2 5.1 and 0.83. Prices per million tokens come from the [AWS models research](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md): Claude Haiku 5.5 through EU geo inference at $0.11 input and $0.55 output, Gemma 4 31B in Frankfurt at $0.168 and $0.48, Claude Haiku 4.5 through EU geo inference at $1.10 and $5.50. All three process in the EU as ADR-0014 requires. Everything below except the first row's time is an estimate, not a measurement.

| Graph | Built by | Model cost for one build | Time | Update |
|---|---|---|---|---|
| Link graph with Markdown links, headings and front matter | Parser in the glue ingestion job | $0 | 0.04 seconds for the 128 files, measured with a regular-expression script | Re-parse changed files |
| Term graph from a confirmed glossary | Lemma and string matching over all chunks | $0 | Not measured | Re-match changed chunks. Re-match everything when the glossary changes |
| Glossary candidates for a human to confirm | One language model pass over the Markdown | About $0.04 with Gemma 4 31B | Minutes | Re-run on new documents |
| LLM-built graph over the Markdown, LightRAG | Language model | $0.34 with Haiku 5.5, $0.42 with Gemma 4 31B | About 13 minutes | Re-extract changed documents at the same price per token |
| LLM-built graph over the Markdown, HippoRAG 2 | Language model | $0.23 with Haiku 5.5, $0.29 with Gemma 4 31B | Not estimated | Skips passages it already indexed |
| LLM-built graph over Markdown, issues and pull requests, LightRAG | Language model | $2.45 with Haiku 5.5, $2.99 with Gemma 4 31B | Not estimated | As above |
| Bedrock GraphRAG over the Markdown | Claude Haiku 4.5, the newest model the feature allows | About $3.42 if it uses as many tokens as LightRAG, plus $424 a month for the smallest Neptune Analytics graph | Unknown | Managed |

The calculations:

- LightRAG over the Markdown: 0.23 × 7.2 = 1.656 million input tokens and 0.23 × 1.26 = 0.290 million output tokens. With Haiku 5.5 that is 1.656 × $0.11 + 0.290 × $0.55 = $0.18 + $0.16 = $0.34. With Gemma 4 31B it is 1.656 × $0.168 + 0.290 × $0.48 = $0.28 + $0.14 = $0.42. With Haiku 4.5 it is 1.656 × $1.10 + 0.290 × $5.50 = $1.82 + $1.59 = $3.42.
- HippoRAG 2 over the Markdown: 0.23 × 5.1 = 1.173 million input and 0.23 × 0.83 = 0.191 million output tokens. With Haiku 5.5 that is $0.13 + $0.10 = $0.23. With Gemma 4 31B it is $0.20 + $0.09 = $0.29.
- LightRAG over Markdown, issues and pull requests: 1.65 × 7.2 = 11.88 million input and 1.65 × 1.26 = 2.08 million output tokens. With Haiku 5.5 that is $1.31 + $1.14 = $2.45. With Gemma 4 31B it is $2.00 + $1.00 = $2.99.
- Glossary candidates: the Markdown read once is 0.23 million input tokens. I assume 200 candidate terms of 50 tokens each, so 0.01 million output tokens. With Gemma 4 31B that is 0.23 × $0.168 + 0.01 × $0.48 = $0.04.
- LightRAG time: 0.23 million tokens in chunks of 1,200 tokens are about 190 chunks, and one extraction plus one gleaning pass per chunk gives about 380 calls. With LightRAG's default of 4 parallel calls and an assumed 8 seconds per call, that is 380 × 8 / 4 = 760 seconds. The 8 seconds is the assumption of the second graph research, not a measurement.
- The Neptune Analytics price and the model list of Bedrock GraphRAG come from the second graph research.

The synthetic workshop results add little. The test dataset research plans about 12 workshops, a third of them with a transcript of 3,000 to 6,000 words. By my estimate that is under 0.05 million tokens.

Money does not separate the options. An LLM-built graph over the Markdown costs less than a dollar, and one over all prose sources about $3. The differences that matter are quality on each question type and what must be rebuilt when a document or the glossary changes.

## Standing architecture questions

1. Retrieval per source type. Prose is found by hybrid retrieval over heading chunks, with the heading path and title in the chunk text. The link graph expands a hit to the supersession chain of a decision record and to the items it references, as ADR-0003 and ADR-0005 already say. No evidence supports graph traversal as the first step for prose. Agent instruction files are prose with their own `source_type` value and a path scope.
2. Scale. The extended link graph for the test dataset adds 1,488 heading nodes and a few hundred edges to the roughly 8,000 nodes the second graph research counted, which already include the 128 Markdown files. A term graph adds one edge per term mention. Generic terms would dominate it, as the "account" count shows, so the build should drop or down-weight terms that occur in a large share of all chunks. No study covers decision records in the hundreds or thousands.
3. Knowledge organization. One index, as the knowledge organization research recommends. The findings add chunk metadata: heading path, outgoing link targets with anchor text, glossary term ids, and for instruction files the path scope. A term found in a question can act as the soft filter that research describes.
4. Embedding independence. The link graph and the term graph use no embedding model and survive a model change untouched. LightRAG, HippoRAG 2 and the SAP dependency graph all embed their entities, so their graphs must be re-embedded with the chunks.
5. Graphs. The question types that need a graph are current decision, decision history and decision to issue to code. The link graph answers them without a language model and without a graph database. A term graph for "where is this domain term defined, decided and implemented" is a candidate with no published measurement. An LLM-built graph stays a contrast arm.
6. Location. The link graph and the term graph run inside the glue ingestion job and call no model, so the model policy does not apply to them and they move on-device unchanged. A glossary candidate pass and any LLM-built graph send document text to a Bedrock route that processes in the EU. Bedrock GraphRAG must use an EU geo inference profile, not a Global one (second graph research).
7. Measurement. See the next section.

## What the bake-off should measure

Measure the documentation graphs as add-ons to one fixed prose retrieval setup, not as new stacks. Each step changes one thing, and every variant gets the same number of context tokens as the baseline.

| Step | Change | What it tests |
|---|---|---|
| D0 | Hybrid retrieval over heading chunks, no structure | Baseline |
| D1 | Heading path and document title added to the chunk text | Structure as text, no graph |
| D2 | D1 plus link graph expansion by one hop: supersession chain, Markdown links, issue and pull request references | Whether following explicit links beats spending the same tokens on more top hits |
| D3 | D2 plus the term graph: chunks tagged with glossary term ids, question terms matched, hits expanded or boosted across decision records, documents, workshop results and code | The owner's hypothesis for domain knowledge, without a language model |
| D4 | The LightRAG and HippoRAG 2 arms the first graph research already put in the head-to-head, scored on the same documentation questions | The contrast with an LLM-built graph |
| D5 | Agent instruction files indexed as ordinary Markdown, against tagged with `source_type` and labeled in the prompt, against left out | Whether the source type changes any answer |

Gold question types that would show a difference, next to the six types the first graph research asked for:

- Decision history: "list every decision about the database layer in order" or "what did WD-004 replace and why". VersionRAG's version listing questions show the largest gap any study in this note reports, 20% against 90%.
- Domain term across sources: "where is the term X defined, which decision changed its rules and which code implements it". This needs a glossary term, a decision record and code that share the term, so it needs the synthetic domain documents.
- Cross-language term: a German question about a term that the sources use in English, and the reverse. Only a glossary with labels in both languages links them by rule.
- Link following: the answer sits in a document that the best hit links to and that shares little vocabulary with the question, such as an ADR that links a technical design.
- Instruction file: "what must be true before a commit in module X" and "which rules apply to this path".
- Stale instruction: an instruction file that disagrees with the code or a newer decision record. A correct answer cites both and says they disagree.
- Single fact and unanswerable questions stay in as a guard. Every graph variant must not lose there, since that is where LLM-built graphs lost in the earlier research.

Sample size limits what the bake-off can show. The evaluation research found that 50 gold questions detect a difference of about 12 points. By the same square-root scaling, 10 questions of one type detect about 27 points and 5 questions about 38 (12 × √(50/10) and 12 × √(50/5), my calculation). Only an effect the size of VersionRAG's would show with 5 questions per type. The new types need at least 10 questions each, or the result is a direction and not a decision.

One test belongs in the quality gates and not in the answer scores. It retrieves a chunk of an instruction file that contains a command, and it passes when the answer quotes the command and fails when the answer model obeys it.

## Findings that touch other tickets

- [Review and adopt: test dataset](https://github.com/Niko4417/OPT-cc/issues/23) and [Write the example questions and the gold question set](https://github.com/Niko4417/OPT-cc/issues/11). The test dataset has no domain knowledge, no glossary, one `AGENTS.md` and no skills. The companion repository needs synthetic domain documents and a glossary in German and English, and optionally a nested `AGENTS.md` and a skill, or four of the question types above cannot be written. The gold set needs the new type tags and at least 10 questions per new type.
- [Review and adopt: graph-based knowledge](https://github.com/Niko4417/OPT-cc/issues/25) and [Review and adopt: graphs and graph databases with a cloud provider](https://github.com/Niko4417/OPT-cc/issues/37). Their recommendations stand. This note adds Markdown links and headings to the link graph and names the term graph as a candidate to measure.
- [Review and adopt: knowledge organization and routing](https://github.com/Niko4417/OPT-cc/issues/38) and [Model the sources around repositories and project artifacts](https://github.com/Niko4417/OPT-cc/issues/29). The `source_type` list of ADR-0018 has no value for agent instruction files, and the chunk metadata has no field for a path scope, link targets or glossary term ids. A glossary that a consultant maintains would be a new kind of project artifact with a prescribed format, like the workshop result format of ADR-0002.
- [Decide how coding agents work on a project memory](https://github.com/Niko4417/OPT-cc/issues/31). The app's own `AGENTS.md` and skill in the project memory folder should be kept out of the index. The stale-file study suggests one sentence for that `AGENTS.md`: where this file and the records disagree, the records win.
- [Review and adopt: quality gates and CI](https://github.com/Niko4417/OPT-cc/issues/24). Two checks without a language model fall out of the link pass: a report of relative links whose target is missing (19 in the test repository) and the instruction-file command test above.
- [Review and adopt: evaluation metrics and protocol](https://github.com/Niko4417/OPT-cc/issues/22) and [Measure the shortlisted stacks head to head on the gold set](https://github.com/Niko4417/OPT-cc/issues/13). Graph variants must be compared at equal context tokens, and steps D0 to D5 are add-ons to one setup, not new arms.
- [Review and adopt: folder and document ingestion](https://github.com/Niko4417/OPT-cc/issues/26). YAKE, the keyword extractor of the one German structure-graph study, is AGPL-3.0 and fails the license rule.

## Open points

- No study measures a glossary-based term graph, in any language. Step D3 would be the first measurement, and its result depends on the quality of a glossary nobody has written yet.
- No study measures any LLM-built graph against chunk retrieval on German documents.
- No study measures graphs or retrieval over large numbers of decision records.
- The time to match a glossary against all chunks with lemmatization is not measured. Neither is the miss rate on German compounds.
- My link counts come from a regular-expression script, not a CommonMark parser. Eight of 61 heading anchors did not resolve under my rule for building anchors, which may be the rule and not the links.
- The Lettria numbers and the heading-path study are read from snippets and an abstract only.

## Method

I read issue #49 and its linked comment, the map in issue #1, `GLOSSARY.md` as of commit `6fe982b`, ADRs 0001, 0003, 0005, 0013, 0016, 0017, 0018 and 0019, and both earlier graph research files. I delegated three research passes that ran in parallel through web search and page fetches. One covered graph results for domain and enterprise documents and German text, one covered graphs built without a language model and link and structure graphs, and one covered agent instruction files. Each pass read papers at arXiv or the publisher, repositories through the GitHub API and vendor documentation, and reported what it could not open. I then re-opened 21 sources whose numbers carry the conclusions and checked the cited values. These are the GraphRAG-Bench medical set, Xiao et al., SAP, LinearRAG, KET-RAG, LinkedIn, VersionRAG, HoH, Document GraphRAG (text extracted from the publisher's PDF), the German statute paper, LARAG, the IBM intranet study, the abstracts of TransArC and of the heading-path study, and the instruction file studies by Chatlatanagulchai, Gloaguen, Galster, Treude and Baltes, Xu and Wu, Yang and Gadgil. I did not re-open the TransArC and LiSSA tables, Table 9 of GraphRAG-Bench, the 2.40 value of the statute paper, Ahmad et al., Karkout et al., Sarmah et al., Capozzi and Helbing, Asai et al., Chinthareddy or the vendor papers by data.world and Writer. Their numbers come from one reading of the paper. I checked the Graphify code comment, the codebase-memory-mcp README and the YAKE license text myself through the GitHub API.

Two numbers differed between the two readings and are left out: the harmful-answer rate of Qwen 2.5 7B in HoH, and the baseline pass rate in SkillSeek's body ablation, of which only the direction is cited.

For the test dataset I made a shallow clone of `thunderbird/thunderbird-android` at commit `1daaefb`, checked out the Markdown files, and ran a Python script that reads inline and reference links, ATX headings and backticked names by regular expression. It resolves relative links against the file tree, turns headings into anchors by lowercasing, dropping punctuation and replacing spaces with hyphens, and matches backticked names against paths, Gradle module notation and Kotlin and Java file names. I ran no indexing job and called no model.

Searches that found nothing: a graph RAG result on WixQA, TechQA, Confluence, Jira, product manuals or API documentation beyond the LinkedIn paper; any evaluation of an LLM-built graph on German documents (queries included "GraphRAG German documents evaluation", "GraphRAG deutsch Evaluation Wissensgraph", "LightRAG German language entity extraction quality", "multilingual GraphRAG entity extraction German benchmark"); glossary or controlled-vocabulary linking measured against LLM extraction; Markdown relative links, heading anchors or front matter as graph edges; and any measured effect on question answering of indexing or boosting agent instruction files.

## Sources

Independent benchmarks and studies:

- Xiang et al., When to use Graphs in RAG (GraphRAG-Bench), medical set: https://arxiv.org/abs/2506.05690
- Xiao et al., GraphRAG-Bench: Challenging Domain-Specific Reasoning: https://arxiv.org/abs/2506.02404
- Ahmad et al., Benchmarking Vector, Graph and Hybrid RAG Pipelines for Open Radio Access Networks: https://arxiv.org/abs/2507.03608
- Ouyang et al., HoH: A Dynamic Benchmark for Evaluating the Impact of Outdated Information on RAG: https://arxiv.org/abs/2503.04800
- Kuehlkamp et al., Does Document Structure Help Dense Retrieval? (abstract only): https://arxiv.org/abs/2610.10170
- Ortmann et al., Evaluating Off-the-Shelf NLP Tools for German, KONVENS 2019, result tables: https://github.com/rubcompling/konvens2019
- Chatlatanagulchai et al., Agent READMEs: https://arxiv.org/abs/2511.12884
- Gloaguen et al., Evaluating AGENTS.md (ETH Zurich and LogicStar.ai): https://arxiv.org/abs/2602.11988
- Galster et al., Configuring Agentic AI Coding Tools: https://arxiv.org/abs/2602.14690
- Treude and Baltes, Context Rot in AI-Assisted Software Development: https://arxiv.org/abs/2606.09090
- Xu and Wu, Stale Instruction Files Override the Code in Coding Agents: https://arxiv.org/abs/2610.30571
- Yang et al., SkillSeek: https://arxiv.org/abs/2609.38822
- Gadgil et al., Bad Memory: https://arxiv.org/abs/2607.14611

Results by a method's own academic authors:

- Zhuang et al., LinearRAG: https://arxiv.org/abs/2510.10114
- Huang et al., KET-RAG: https://arxiv.org/abs/2502.09304
- Huwiler et al., VersionRAG: https://arxiv.org/abs/2510.08109
- Knollmeyer et al., Document GraphRAG, Electronics 14(11), 2102: https://www.mdpi.com/2079-9292/14/11/2102
- Siddig et al., Verifiable Retrieval for Legal Reasoning through Structure-driven Knowledge Graph: https://arxiv.org/abs/2610.13695
- Capozzi and Helbing, Agentic Graph RAG for Auditable Commercial Registry Analysis: https://arxiv.org/abs/2605.18770
- Karkout et al., Efficient KG-Augmented RAG with Reusable Graph Community Summaries: https://aclanthology.org/2026.kallm-1.12/
- Keim et al., Recovering Trace Links Between Software Documentation And Code (TransArC), ICSE 2024: https://publikationen.bibliothek.kit.edu/1000165692
- Fuchß et al., LiSSA, ICSE 2025: https://fuchss.org/assets/pdf/2025/icse-25.pdf
- Asai et al., Learning to Retrieve Reasoning Paths over Wikipedia Graph: https://arxiv.org/abs/1911.10470
- Chinthareddy, Reliable Graph-RAG for Codebases: https://arxiv.org/abs/2601.08773

Company and vendor results:

- Min et al. (SAP), Towards Practical GraphRAG: https://arxiv.org/abs/2507.03226
- Xu et al. (LinkedIn), RAG with Knowledge Graphs for Customer Service Question Answering: https://arxiv.org/abs/2404.17723
- Sarmah et al. (BlackRock, NVIDIA), HybridRAG: https://arxiv.org/abs/2408.04948
- Bolognesi et al. (Rulex, University of Genoa), LARAG: https://arxiv.org/abs/2605.07517
- Fagin et al. (IBM), Searching the Workplace Web, WWW 2003: https://mccurley.org/papers/www2003/p641-mccurley.html
- Sequeda et al. (data.world): https://arxiv.org/abs/2311.07509
- Mozolevskyi and AlShikh (Writer): https://arxiv.org/abs/2405.02048
- Lettria on the AWS Machine Learning blog (unverified): https://aws.amazon.com/blogs/machine-learning/improving-retrieval-augmented-generation-accuracy-with-graphrag/
- Pillar Security, Rules File Backdoor, 2025-03-18: https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents

Formats, guidance and tool documentation:

- AGENTS.md: https://agents.md
- Claude Code memory, skills and subagents: https://code.claude.com/docs/en/memory, https://code.claude.com/docs/en/skills, https://code.claude.com/docs/en/sub-agents
- Agent Skills specification: https://agentskills.io/specification
- Codex AGENTS.md guide: https://developers.openai.com/codex/guides/agents-md
- GitHub Copilot repository instructions: https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions
- Cursor rules: https://cursor.com/docs/context/rules
- OWASP LLM01:2025 Prompt Injection: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- LightRAG `env.example`: https://github.com/HKUDS/LightRAG/blob/main/env.example
- Microsoft GraphRAG auto prompt tuning: https://microsoft.github.io/graphrag/prompt_tuning/auto_prompt_tuning/
- Graphify `graphify/cluster.py`: https://github.com/Graphify-Labs/graphify/blob/HEAD/graphify/cluster.py

Tool repositories: [spaCy](https://github.com/explosion/spaCy), [simplemma](https://github.com/adbar/simplemma), [pyahocorasick](https://github.com/WojciechMula/pyahocorasick), [CharSplit](https://github.com/dtuggener/CharSplit), [HanTa](https://github.com/wartaal/HanTa), [YAKE](https://github.com/LIAAD/yake), [markdown-it-py](https://github.com/executablebooks/markdown-it-py), [LlamaIndex](https://github.com/run-llama/llama_index), [codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp), [graph-rag](https://github.com/datastax/graph-rag), [obsidiantools](https://github.com/mfarragher/obsidiantools), [basic-memory](https://github.com/basicmachines-co/basic-memory).

Test dataset: `thunderbird/thunderbird-android` at commit [`1daaefb`](https://github.com/thunderbird/thunderbird-android/commit/1daaefba6a2994496ec2e3a73888584ec7f00f54).

Earlier research in this repository, on their `research/*` branches: `graph-knowledge.md`, `graph-knowledge-v2.md`, `aws-models.md`, `test-dataset.md`, `coding-agents.md`, `knowledge-organization.md` and `local-evaluation.md`.
