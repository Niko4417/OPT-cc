# How much classification, parsing and chunking change answer quality

Date: 2026-11-20. Ticket: [#52](https://github.com/Niko4417/OPT-cc/issues/52). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1). Context: [ADR-0017](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0017-feed-the-managed-retrieval-arm-our-own-chunks-and-score-it-per-source-type.md), [ADR-0018](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0018-ingest-github-repositories-with-our-own-glue-job-into-our-own-chunk-store.md), [ADR-0019](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0019-split-repository-files-by-type-skip-by-content-and-cite-commit-pinned-permalinks.md), [ADR-0020](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0020-sync-github-repositories-and-embeddings-incrementally.md). These are research findings, not owner decisions.

Every source was read on 2026-11-20. Numbers in square brackets point to the list under "Sources", where each entry has its link. "Points" means percentage points of the named metric. "Vendor" marks a number that the maker of the product measured.

## Question

How much do the choices made at ingestion change answer quality, and does a language model at ingestion pay off? Ingestion means deciding a file's type, parsing it and cutting it into chunks. Today no language model takes part. Rules decide the type, with go-enry and Magika as second checks (ADR-0019). Code is cut along its syntax tree, Markdown along headings and documents by Docling (ADR-0018). The ticket asks six things: the effect of chunking per source type, the effect of parsing errors, the gain and cost of a language model at ingestion, whether a second model lowers errors, which part of a pipeline moves results most, and which extra runs would measure all this in the bake-off.

## Short answer

- Chunking changes results by 1 to 6 points once it follows the structure of the source and stays within about 250 to 1,000 tokens per chunk. Outside that range the loss is large. Chunks of 64 tokens against 1,024 tokens differed by 57 points of recall@1 on technical documents, and in the opposite direction by 26 points on short fact questions [13].
- Semantic and model-based chunkers showed no consistent gain over structural or fixed-size chunks in the independent tests found [2, 14, 18, 21]. The gains their authors report are 2 to 7 points on narrative books and on a benchmark built for the method [9, 19, 20].
- Parsing errors cost more than chunking on tables and scans. The best text recognition system lost 5.0 points of end-to-end F1 against perfect text, and an older one lost 21.6 [8]. On financial tables, the parsers differed by 20 points of exact match [15]. On clean digital PDFs the parsers differed by about 6 points of retrieval MRR [14].
- A model-written context line per chunk gained 20 points on test sets built from chunks that cannot be understood alone [11], and 0 to 3 points in independent tests on ordinary document sets [10, 12, 30, 52]. Anthropic's own figure is a 35% lower retrieval failure rate, which is 2.0 points of recall@20 [1]. In the one controlled comparison, the real heading path of a chunk, which costs nothing, did better than the model-written line [12].
- A context line for 10,000 to 50,000 chunks costs about 3 to 73 dollars once on Bedrock in the EU, with Claude Haiku 5.5 or a Gemma 4 model, depending on how much of the document the model reads. A repository of 1 to 2 million lines costs about 11 to 31 dollars (estimates, section 3).
- Two models that agree are right more often than one model alone, and a disagreement marks the items worth a human look. A second model does not catch the errors both share, and models share many of them [39, 42]. No study found tests a second model for file classification.
- In every study that varied several parts of one pipeline, the retrieval method, the reranker or the answer model moved results more than chunking did, with one exception that used a 100-character chunk as its baseline [14, 15, 17, 24, 25, 27, 28, 29, 30].
- German evidence is thin. One study of the German Civil Code found chunks along sections and subsections best and model-based methods worse [21]. No study of German parsing errors, German transcripts or German issue threads was found.

## Measured effects, sorted by size

Rows are sorted by the largest absolute effect in the row, whatever its sign. The second column is the difference in points. The row for semantic chunking is placed by its result on real documents, because its larger figure comes from artificial documents.

| Choice varied | Effect in points | Metric | Who measured it, on what | Language and type |
|---|---|---|---|---|
| Chunk size, 64 against 1,024 tokens | +57.1 for large chunks on TechQA (4.86 to 61.92), -25.6 on SQuAD (64.19 to 38.55) | recall@1 | Bhat et al., Fraunhofer IAIS, six QA sets, two embedding models, no overlap [13] | English prose, four sets stitched into long documents |
| Retriever family | 49.0 on narrative pages (BM25 0.210, dense E5 0.700), 16.7 on table pages (BM25 0.677, ColBERT 0.844) | page MRR | El Bachyr et al., University of Luxembourg and BGL BNP Paribas, FinanceBench and their TableQuest [14] | English financial PDFs |
| Transcript errors | -29.8 (54.47 to 24.67) at 33.6% word error rate and 52.4% speaker error rate | answer accuracy | Tang et al., Soochow University, their own meeting QA benchmark [33] | English and Mandarin meetings |
| Answer model | 28.9 between the best and worst of 8 models on financial tables, 19.1 on average over six sets | exact match and F1 | Hui et al., UDA benchmark [15] | English PDFs with tables |
| Late chunking | +1.5 to +1.9 (vendor) [3]. -21.7 on COVID-QA [11]. -12.7 on MS MARCO [10] | nDCG@10, nDCG@5 | Jina AI, Conti et al., Merola and Singh | English prose |
| Text recognition | -5.0 for the best system (36.1 to 31.1), -6.1 for MinerU, -21.6 for Nougat | end-to-end F1 | Zhang et al., OHRBench, 8,561 pages, 8,498 questions [8] | English and Chinese documents |
| Context line on context-dependent chunks | +20.4 (52.0 to 72.4), and 0.0 on a standard set (63.2 to 63.2) | nDCG@10 | Conti et al., ConTEB, eight sets, context written by Qwen2.5-7B [11] | English prose |
| Parser choice | 20.0 between best and worst of four parsers on financial tables (72.4 to 52.4), 5.7 on tables in papers | exact match and F1 | Hui et al., UDA benchmark, 341 and 100 questions [15] | English PDFs with tables |
| Spoken documents | -19.1 at 22.7% word error rate, average of five QA models | F1 | Li et al., Spoken SQuAD [34] | English, synthetic speech, models of 2018 |
| Proposition chunks | +12.0 and +9.7 with unsupervised retrievers, +2.6 and +2.8 with supervised ones [4]. -18.8 against recursive chunks on books [20] | recall@5, recall@20 | Chen et al., Dense X Retrieval, and Duarte et al. | English Wikipedia and English books |
| Overlap, 0% against 25% | +12.9 and +9.8 (El Bachyr, one chunker) [14]. +1.5 (Microsoft, vendor) [22]. -1.4 (Chroma, practitioner report) [9]. Often not significant (IBM) [25] | MRR, recall@50, recall | four sources | English prose and tables |
| Reranker after hybrid search | +12.1 (0.695 to 0.816) | recall@5 | Akarsu et al., T2-RAGBench, 23,088 questions [30] | English financial text with tables |
| Questions per chunk | +10.7 on SQuAD, +5.1 on Natural Questions, down to -5.0 on multi-hop questions with some embedding models | top-1 context accuracy, full match@20 | Neeser et al., QuOTE [36] | English prose |
| Chunking method | 8.3 between best and worst of 13 settings (83.6 to 91.9) | token recall | Smith and Troynikov, Chroma, practitioner report, 472 generated questions [9] | English prose, chat logs, reports |
| Model-written metadata in front of the chunk | +3.5 to +7.2 | answer F1 (RAGChecker) | Dadopoulos et al., FinanceBench, 150 questions [29] | English financial filings |
| Parser choice on digital PDFs | 5.8 on narrative pages (0.588 to 0.646), 4.9 on table pages (0.758 to 0.807) | page MRR | El Bachyr et al., six parsers [14] | English financial PDFs |
| Chunking method on digital PDFs | 5.8 on narrative pages (0.600 to 0.658), 3.9 on table pages (0.794 to 0.833) | page MRR | El Bachyr et al., six chunkers [14] | English financial PDFs |
| Parser output used unchanged | -4.8 (62.1 to 57.3) after switching to Docling's HTML output | share of correct answers | Mankour et al., Climate Finance Bench, 330 questions [28] | English sustainability reports |
| Syntax-tree chunks of code | +4.3 | recall@5 | cAST, already in the GitHub ingestion research [53] | Code |
| Context line, vendor | +2.0 (94.3 to 96.3), which is the reported 35% lower failure rate. +4.2 on code (90.06 to 94.29) | recall@20, pass@20 | Anthropic, on codebases, fiction and papers [1, 50] | English prose and code |
| Chunk boundaries at sections | +4.1 and +1.9 against fixed windows of equal size | coverage nDCG@10 | Kuehlkamp et al., University of Notre Dame, preregistered, two corpora [12] | English Wikipedia and research papers |
| Semantic against fixed-size chunks | -3.2 to +1.6 on real documents, up to +20.1 on documents stitched from unrelated texts | F1@5 | Qu et al., two of three authors at Vectara [2] | English prose |
| Context line, independent tests | +2.9 and +1.6 [12]. +2.8 [30]. +0.9 and +2.6 [10]. 0 [52] | coverage nDCG@10, recall@5, nDCG@5, hit@5 | four independent sources | English prose and filings |
| Summary tree over chunks (RAPTOR) | +0.5 to +2.2 | F1 and accuracy | Sarthi et al., the method's authors [7] | English papers and stories |
| Heading path in front of the chunk | +1.0 and +0.8 over section chunks. A shuffled, wrong path gave 0.0 and -0.8 | coverage nDCG@10 | Kuehlkamp et al. [12] | English prose |

## 1. Chunking per source type

### Prose documents

**Chunk size.** The best size depends on where the answer sits, and a wrong size costs more than any choice of method.

- Bhat et al. tested 64 to 1,024 tokens on six sets with two embedding models. Short fact questions (SQuAD) did best at 64 tokens. Technical and narrative documents did best at 512 to 1,024 tokens. The gap between the best and worst size was 6.5 to 67.1 points of recall@1, and the best size changed with the embedding model on three of six sets [13].
- Wang et al. found faithfulness of 97.59 at 512 tokens, between 94.26 and 97.22 at 128, 256 and 1,024 tokens, and 80.37 at 2,048 tokens, on one financial filing with about 170 generated questions [24].
- Microsoft measured recall@50 of 42.4 at 512 tokens, 37.5 at 1,024 and 34.9 at 8,191 on its customer sets (vendor) [22].
- NVIDIA measured answer accuracy between 0.603 and 0.645 for token chunks of 128 to 2,048 tokens and 0.648 for one chunk per page, over five sets. On one set, 128-token chunks scored 0.421 against 0.520 for pages (vendor) [23].
- Chroma found recall almost flat between 200 and 800 tokens (85.4 to 89.5) while precision rose from 1.5% to 7.0% as chunks shrank [9].
- IBM varied 256, 384 and 512 tokens across five sets and found marginal differences below 0.02 of answer correctness [25]. Li et al. report only minor differences across their chunk sizes [26].
- Unstructured measured 29.08% correct answers at 128 tokens, 41.84% at 512 tokens and 43.97% for chunks built from document elements, judged by GPT-4 on 141 FinanceBench questions (vendor) [16].

**Overlap.** The results disagree. El Bachyr et al. found +12.9 points of MRR from 0% to 25% overlap and no further gain at 50%, with one model-based chunker [14]. Microsoft found +1.5 points from 25% overlap over chunks cut at sentence boundaries (vendor) [22]. Singh found 32 tokens of overlap better than 0 or 64 at each size, 0.90 against 0.87 recall@5 [31]. Chroma found that overlap lowered recall with a strong embedding model (88.1 against 89.5) and raised it with a weak one (82.4 against 77.1) [9]. IBM found overlap often not significant [25]. The late chunking paper found no clear advantage for overlap [3].

**Method.**

- Structural boundaries beat fixed windows of equal size by 4.1 and 1.9 points in a preregistered test on 951 and 4,303 questions. The authors rank the step to structural boundaries as two to four times the size of the step that adds a heading path [12].
- Semantic chunking, which cuts where the embedding of neighbouring sentences changes, showed no consistent gain. Qu et al. found it within -3.2 and +1.6 points of fixed-size chunks on real documents, at most +1.6 on evidence retrieval and at most 0.01 BERTScore on answers. Its gains of up to 20 points came from documents stitched together from unrelated texts [2]. El Bachyr et al. measured semantic chunks 4.5 points of MRR below sentence chunks [14]. Chroma's lowest recall belongs to a semantic chunker (83.6) and its second highest to another one (91.3) [9]. Kreileder et al. found a cluster-based chunker worst on 13 theses, with 10 questions and an unreliable judge [18].
- Chunkers that ask a language model where to cut gained 7.4 points of DCG@20 over recursive chunks on 100 English books, according to the method's authors, and took 95 to 1,628 seconds per book against under 1 second [20]. Chroma measured +2.4 points of recall for a GPT-4o chunker over recursive chunks of similar size [9]. Tencent's HiChunk gained 7.0 points of evidence recall on its own benchmark and little on existing ones, which its authors explain by questions whose evidence is only a few sentences long [19]. On the German Civil Code the same kind of chunker landed in the worst group and took 9 to 12 hours to build [21].
- Hierarchical methods. RAPTOR, a tree of model-written summaries, gained 0.5 to 2.2 points in its authors' controlled comparison [7]. A two-stage search that first picks sections and then chunks lost 3.3 and 1.5 points against flat search [12], which matches the routing results in the knowledge organization research [53].
- Late chunking embeds the whole document and pools per chunk afterwards. Jina AI reports +1.5 to +1.9 points of nDCG@10 on four sets (vendor) [3]. Conti et al. measured +9.0 on average over context-dependent sets and -21.7 on COVID-QA [11]. Merola and Singh measured -12.7 on MS MARCO with one model, and a fall from 0.246 to 0.070 nDCG@5 with BGE-M3 on NFCorpus [10]. The method ties the index to one long-context embedding model.
- Proposition chunks, short stand-alone statements written by a model, gained 9.7 to 12.0 points of recall@5 with untrained retrievers and 2.6 to 2.8 with trained ones, and 2.7 to 4.1 points of exact match in answers. The index grew from 41.4 million passages to 256.9 million propositions [4]. On narrative books they scored 18.8 points below recursive chunks [20].
- One outlier reports a spread of 21.5 points of nDCG@5 across 36 chunking strategies against 4.0 between two embedding models. Its weakest strategy is a window of 100 characters, and a model judged relevance without seeing the question [17].

### Workshop results

No study found measures chunk size, overlap or chunking method on workshop minutes or meeting transcripts. Two related results exist.

- Tang et al. index each utterance as one unit with its start time, end time and speaker. Their text baseline fell from 54.47% to 24.67% accuracy when transcript and speaker labels went from ground truth to a 33.6% word error rate and a 52.4% speaker error rate [33]. They did not vary the chunking.
- Chroma's test set includes one chat-log corpus, but the part of the report read for this note gives no result per corpus [9].

The turn lines with time ranges that ADR-0002 prescribes match the unit Tang et al. use. Whether single turns or windows of several turns retrieve better is not measured anywhere I found.

### Issue and pull request threads

No study found compares one record per comment with one record per thread, or measures chunk size on issue trackers. The nearest evidence concerns short records that look alike.

- On legal contracts that resemble each other, more than 95% of retrieved chunks came from the wrong document. A document summary of about 150 characters in front of each chunk roughly halved that rate, according to the authors, who give exact values only in figures [37].
- BM25 recall@10 on tables fell from 0.967 to 0.592 when the title was removed [32].

Both support the header that ADR-0019 already puts on every GitHub record (repository, number, title, state, labels). Neither tests it on issue threads.

### Tables

- How a table is represented decides retrieval. Dense retrieval over the whole table reached recall@10 of 0.741 on FeTaQA, column names alone 0.208, a model-written table summary 0.436 and BM25 0.082. One vector per row scored within 3 points of one vector per table at 270 to 6,000 times the query time [32].
- Table questions need a different retriever than narrative questions. ColBERT led on table pages (MRR 0.844) and a dense model on narrative pages (0.700). BM25 was last on both [14]. On financial documents that mix text and tables, BM25 beat a dense model (recall@5 0.644 against 0.587), and the reranked hybrid reached 0.816 [30].
- Parsing matters most here, see section 2.
- Chunks built from document elements with a prefix and a table description reached 53.19% correct answers against 48.23% for 512-token chunks, in manual grading of 141 questions (vendor) [16].
- NVIDIA kept tables and charts as whole units in all its runs and did not test splitting them (vendor) [23]. No controlled test of splitting a table by rows with a repeated header was found, only forum posts.

### German and other languages

- Prior et al. (Technical University of Munich) tested 21 chunking strategies on the German Civil Code with 525 German questions. Subsections reached recall@10 of 0.47 and sections 0.46. Sentences and propositions followed. Context lines written by a model were "competitive but clearly below" the structural units. Semantic clustering and RAPTOR were in the middle. The model-based chunker and fixed windows were worst. Build time was 51 seconds for sections, 38 to 52 minutes for context lines, 3 to 6 hours for RAPTOR and 9 to 12 hours for the model-based chunker [21].
- Microsoft's hybrid search numbers include German as one of 18 languages in an average (vendor) [22]. Li et al. added French and German documents to an English knowledge base and saw slightly lower scores [26].
- A Portuguese study of one book found a sentence-window strategy worse than plain 1,000-token chunks (5.77 against 6.04 of 10) [27].
- A hospital study with mostly German documents exists but its page refused access, so its numbers are not in this note [54].

## 2. Parsing quality

- OHRBench compares retrieval and answers over machine-read pages with the same pipeline over human-checked text, on 8,561 pages from seven domains. With perfect text the pipeline scored 36.1 F1 overall and 70.0 on retrieval. The best reader, a 72-billion-parameter vision model, scored 31.1 and 59.2. MinerU scored 30.0 and 50.1, Marker 29.5 and 56.6, and Nougat 14.5 and 40.9. The gap was largest for charts (32.9 to 19.0) and reading order (18.7 to 8.8), and smaller for plain text (45.0 to 40.6) and tables (34.6 to 31.1). Wrong characters hurt more than wrong formatting. The paper tests neither Docling nor German [8].
- UDA ran four parsers on financial reports with tables. With GPT-4 Turbo answering, exact match was 72.4 for a GPT-4o parse, 68.0 for raw text extraction, 61.3 for a layout model and 52.4 for a layout model plus a language model, against 71.9 for hand-parsed tables. On tables in research papers the spread was 5.7 points [15].
- El Bachyr et al. compared six open-source parsers on digital financial PDFs. Page MRR ranged from 0.588 (PyPDF2) to 0.646 (pdfminer) on narrative questions and from 0.758 to 0.807 (pdfplumber) on table questions. Unstructured took 1.888 seconds per page against 0.003 for the fastest parser and ranked third. Answer accuracy was not measured per parser [14].
- Singh crossed three parsers, three chunkers and five embedding models on four Indian government documents. Between 98.3% and 99.8% of the gold evidence survived every parser and chunker combination, so the differences came from ranking. Parser and chunker interacted: the largest coefficient in the model was an interaction (-0.077 recall@10), larger than any single parser or chunker effect [31].
- Climate Finance Bench lost 4.8 points of correct answers when it switched its PDF text to Docling's HTML output without cleaning it, after hybrid search and a reranker had added 7.3 points [28].
- Most et al. compared text recognition followed by text retrieval with retrieval over page images on 4,196 documents in four scan quality classes. With a large vision model as reader, MRR fell from 0.515 on digital documents to 0.452 on badly degraded scans. With Nougat as reader it was between 0.230 and 0.322. The choice of reader moved results more than scan quality did [35].
- Speech. At a 22.7% word error rate, five QA models of 2018 lost 19.1 points of F1 on average [34]. At 33.6% word error rate with 52.4% speaker errors, a 2026 text pipeline lost 29.8 points of accuracy [33]. No study found measures the loss at the 4% German word error rate that the transcription research reports for Parakeet [53].

Nothing found measures Docling's default pipeline end to end, and nothing measures any parser on German documents. The document ingestion research already notes that no German parsing benchmark exists [53].

## 3. A language model at ingestion

### A model-written context line per chunk

Anthropic's write-up of 2024-09-19 exists and says what the owner recalled. A model reads the whole document and one chunk and writes a short line that places the chunk in the document. The line is put in front of the chunk before embedding and before building the BM25 index. Anthropic reports a top-20 retrieval failure rate of 5.7% without it, 3.7% with it (35% lower), 2.9% with the BM25 index also built on the extended text (49% lower) and 1.9% with a reranker on top (67% lower). The figures are averages over codebases, fiction and papers with the best embedding model of the test, Gemini Text 004 [1]. In recall@20 that is 94.3 to 96.3, then 97.1, then 98.1.

Reproductions and independent tests:

| Source | Data | Result |
|---|---|---|
| Pantoja, public reproduction with Anthropic's code and data [50] | 9 codebases, 737 chunks, 248 questions | Baseline reproduced exactly (pass@20 90.06). With context lines 94.99 against Anthropic's 94.29. A second embedding model gained +5.8 |
| DenserAI, vendor of a retriever, same data [51] | same | BM25 recall@20 70.48 to 89.26. Dense 90.06 to 94.48. With a reranker 94.15 to 96.10 |
| Conti et al., ConTEB [11] | Eight sets where chunks need document context | nDCG@10 52.0 to 72.4. No change on a standard set (63.2). Indexing took 1,891 ms per document against 18 ms, with a 7-billion-parameter model on one A100 GPU |
| Kuehlkamp et al., preregistered [12] | 200 Wikipedia articles and 1,585 papers | +2.9 and +1.6 over fixed windows. Below section chunks without any prefix (-1.2 and -0.3) and below section chunks with the heading path (-2.2 and -1.2, both significant). The lines were written by Gemma 4 26B-A4B |
| Akarsu et al. [30] | 7,318 financial documents, 23,088 questions | A model-written document summary in front of each document: recall@5 +2.8 dense, +2.2 hybrid. The reranker added +12.1 |
| Merola and Singh, workshop paper [10] | 50 questions, about 300 documents | nDCG@5 +0.9 and +2.6 with two embedding models. No measurable change in answers |
| Prior et al. [21] | German Civil Code, 525 questions | Below section and subsection chunks, exact value only in a figure |
| Tirth G., informal public test [52] | 15 documents, 88 hand-written questions | No gain at document level (hit@5 100% against 100%, and 96.2% against 100%) |

So the method works as described on Anthropic's data, and the gain is confirmed there by two outside runs. On other data the gain ranges from 0 to 3 points, except where chunks are unreadable without their document, where it is 20 points. Three results say where the gain comes from. BM25 gains most (+18.8 points of recall@20 [51]). The gain shrinks when a reranker follows (2.0 points against 4.4 [51]). A real heading path gives more than a model-written line on documents that have headings [12].

### Summaries, questions and propositions per chunk

- Generated questions per document (doc2query) are an established way to help BM25 [5]. Gospodinov et al. showed that many generated questions cannot be answered from their passage. Keeping only the best 30% raised MRR@10 from 0.279 to 0.323 and cut the index by 33% [6].
- Weller et al. tested eleven expansion methods with 24 retrieval models on twelve sets. Expansion helped weak retrieval models and "generally harms stronger models", because the added text creates false matches [38].
- QuOTE stores each chunk once per generated question. It gained 5.1 to 10.7 points of top-1 accuracy, with 10 to 15 questions per chunk, and lost up to 5.0 points on multi-hop questions with some embedding models [36].
- RAPTOR and propositions are covered in section 1.
- Model-written pages about code belong to [#51](https://github.com/Niko4417/OPT-cc/issues/51) and are left out here.

### Model-extracted metadata

- Dadopoulos et al. had Gemini 2.5 Flash write clusters, entities and answered questions per chunk and put them in front of the chunk text. Answer F1 rose by 3.5 to 7.2 points on FinanceBench. A reranker alone added 8.5 points. Using the same metadata to expand retrieved chunks lowered F1 from 44.4 to 40.3 [29].
- The knowledge organization research already reports the one result on metadata used as a filter, Hits@10 from 0.74 to 0.90 [53].

### Model-based classification of files the rules cannot place

No study found tests a language model for deciding a file's type or programming language. The only measured classifier is Magika, with an average F1 of 99% over more than 100 content types on over 1 million files, reported by its makers at Google [49]. ADR-0019 deferred the model check because the test dataset has no files to measure it on. That reason still holds.

### Cost on Bedrock in the EU

Prices per million input and output tokens come from the models research [53]: Nova Micro 0.046 and 0.184 dollars, Claude Haiku 5.5 0.11 and 0.55, Gemma 4 26B-A4B 0.156 and 0.48, Gemma 4 31B 0.168 and 0.48, Claude Sonnet 5.5 2.20 and 11.00. The token counts below are my estimates. The cost of one pass is chunks times input tokens per chunk times the input price, plus chunks times output tokens per chunk times the output price.

Assumptions: 400 tokens per chunk, 50 tokens of instructions, 100 tokens of output per context line. Anthropic's own example uses 800-token chunks and 8,000-token documents [1]. A line of code is taken as 10 tokens, so 1 million lines are 10 million tokens and 25,000 chunks.

Worked example, context line with the whole document, 10,000 chunks, Claude Haiku 5.5: input is 10,000 × (8,000 + 400 + 50) = 84.5 million tokens, which costs 84.5 × 0.11 = 9.30 dollars. Output is 10,000 × 100 = 1.0 million tokens, which costs 0.55 dollars. Together 9.85 dollars.

| Pass | Input and output tokens per chunk | Chunks | Nova Micro | Haiku 5.5 | Gemma 4 26B-A4B | Gemma 4 31B | Sonnet 5.5 |
|---|---|---|---|---|---|---|---|
| Context line, whole 8,000-token document | 8,450 and 100 | 10,000 | $4.07 | $9.85 | $13.66 | $14.68 | $196.90 |
| | | 50,000 | $20.35 | $49.23 | $68.31 | $73.38 | $984.50 |
| Context line, 2,000-token window around the chunk | 2,450 and 100 | 10,000 | $1.31 | $3.25 | $4.30 | $4.60 | $64.90 |
| | | 50,000 | $6.55 | $16.23 | $21.51 | $22.98 | $324.50 |
| Summary or questions, chunk only | 500 and 150 | 10,000 | $0.51 | $1.38 | $1.50 | $1.56 | $27.50 |
| | | 50,000 | $2.53 | $6.88 | $7.50 | $7.80 | $137.50 |
| Context line for code, 3,000 tokens of the file | 3,450 and 100 | 25,000 (1 million lines) | $4.43 | $10.86 | $14.65 | $15.69 | $217.25 |
| | | 50,000 (2 million lines) | $8.86 | $21.73 | $29.31 | $31.38 | $434.50 |
| Second model checks each context line against the document | 8,600 and 30 | 10,000 | $4.01 | $9.62 | $13.56 | $14.59 | $192.50 |
| | | 50,000 | $20.06 | $48.12 | $67.80 | $72.96 | $962.50 |
| Classify 500 files the rules cannot place | 2,200 and 50 | 500 | $0.06 | $0.13 | $0.18 | $0.20 | $2.70 |

All amounts are estimates in USD without VAT and without prompt caching. Anthropic states 1.02 dollars per million document tokens with prompt caching for its own example [1]. The models research holds no cache prices for Bedrock, so a cached price is not computed here. Three further costs follow.

- A checker pass about doubles the bill, because the checker reads the same document again.
- The extended text is embedded, so each chunk grows by about 100 tokens. At 0.12 dollars per million tokens for Cohere Embed v4 [53], that is 0.12 dollars per 10,000 chunks.
- A context line depends on the whole document. When one section of a file changes, the lines of all its chunks are out of date. The vector cache of ADR-0020 is keyed by chunk text, so the extended text must be the key, and a small edit then re-embeds the whole file. A legacy program of 1 to 2 MB is about 250,000 to 500,000 tokens (estimate) and does not fit a whole-document prompt at all, which is why the code row uses a window.

### When the model-written text is wrong

No study found measures how often a context line is wrong, or what a wrong line does to answers and citations. Related measurements:

- A wrong prefix can cost more than no prefix. Heading paths shuffled between documents lowered the score by 0.8 points on research papers and changed nothing on Wikipedia [12].
- Model-written summaries had minor invented content in 4% of 150 sampled nodes, with no measurable effect on answers, according to RAPTOR's authors [7].
- Of 408 propositions written by GPT-4, 0.7% were not faithful to the passage and 4.9% were not understandable alone [4].
- Generated questions that the passage cannot answer lowered retrieval until they were filtered out [6].
- In entity extraction from long reports, 1.2% of false extractions had no support in the text. Most errors were real mentions outside the wanted scope. A misleading sentence planted in the report was taken as fact in 73% to 100% of cases, by cheap and flagship models alike (single-author preprint) [47].

### How answers still cite the source

The methods keep the link to the source in different ways.

- Anthropic puts the line in front of the chunk and passes the extended chunk to the answer model, "distinguishing between what is context and what is the chunk" [1]. The chunk and its position do not change.
- Dense X retrieves propositions and scores them by the passage each came from [4]. QuOTE removes duplicates by original chunk [36]. Both need a stored link from the generated text to its chunk.
- A RAPTOR summary node covers 6.7 child nodes on average, and 18.5% to 57% of retrieved nodes were summary nodes [7]. Such a node has no single line range to cite.

For a project memory with commit-pinned permalinks (ADR-0019) and a hard gate of 0 fabricated citations, this means model-written text can be a search field of a chunk record and nothing else. A citation points to the chunk's own anchor. A statement that appears only in the model-written line has no source and cannot be cited.

## 4. Two models at ingestion

**Extraction.** Three medical evidence studies compare models, model pairs and humans against a checked reference.

- Khan et al. let GPT-4-turbo and Claude-3-Opus extract 23 variables from 17 trials independently. They agreed on 87% of 391 answers, and the agreed answers were 94% correct. Where they disagreed, GPT-4-turbo was 41% correct and Claude-3-Opus 50%. After each model had seen the other's answer, 51% of the disagreements were resolved, with 76% accuracy [39]. From these figures, a single model was about 87% to 88% correct overall, and 42% to 46% of each model's errors sat in answers where both agreed (my calculation from the abstract).
- Gartlehner et al. compared a model's extraction checked by one human with extraction by humans only, on 9,341 data elements in six running reviews. Accuracy was 91.0% against 89.0%, major errors 2.5% against 2.7%, and the model-assisted way saved a median of 41 minutes per study [40].
- For two humans, Buscemi et al. found that single extraction made 21.7% more errors than double extraction and took 36.1% less time [41].

**Shared errors.** Kim et al. compared the wrong answers of 349 and 71 models on two leaderboards. Two models that were both wrong chose the same wrong answer 60% of the time on one set, where chance is 33%, and 42.3% on the other, where chance is 12.7%. Models of the same company agreed 2.2 to 6.6 points more, and more accurate models agreed more [42]. Goel et al. report the same trend and that judge models favour models similar to themselves [43]. A checker from another model family, as the models research proposes [53], lowers the shared part only a little by these numbers.

**Checker quality.** On LLM-AggreFact, which tests whether a claim is supported by a document, the best checkers reach an average score of 78.5 to 79.5 of 100 and small specialised ones 75.0 to 77.4 [48]. A checker of context lines would itself be wrong on a noticeable share of hard cases.

**Classification, cascades and self-consistency.**

- A cascade sends each item to a cheap model first and to a stronger one only when the first is unsure. FrugalGPT matched GPT-4 with up to 98% lower cost [45]. Ibrahim and Zaki paired a cheap classifier with a large model on 15 English annotation tasks: 0.674 accuracy against 0.678 for the large model alone at 56% of the cost. Items the cheap model rated at 0.9 confidence or more were 81.5% correct and covered 37.6% of items. On one task 78% of items got that rating and only 38.3% were correct [46]. A cascade saves money and depends on a confidence value that can fail per task.
- Self-consistency, the majority of several sampled answers of one model, gained 3.9 to 17.9 points on arithmetic and commonsense reasoning. Its paper does not test classification or extraction [44]. In entity extraction, how often an entity recurred across samples raised the precision of the top 30% by 12 to 23 points, while the confidence a model states was badly calibrated (calibration error 0.59 to 0.88) [47].
- No study found compares one model, two models, rules and a human on file classification. For rules, the one number is Magika's 99% F1 [49].

## 5. Relative weight

Studies that varied several parts of one pipeline:

| Study | Parts varied | Largest effect | Chunking or parsing effect |
|---|---|---|---|
| El Bachyr et al. 2026 [14] | 4 retrievers, 6 parsers, 6 chunkers, 3 overlaps, 12 answer models | Retriever, 49.0 points of MRR. Answer model, 36.7% to 54.0% correct among mid-size and large models | Overlap 12.9, parser 5.8, chunker 5.8 points of MRR |
| Hui et al., UDA [15] | 4 parsers, 5 retrievers, 8 answer models | Answer model, 28.9 on financial tables. Gap between real and perfect retrieval, up to 23.5 | Parser up to 20.0 on financial tables, 5.7 on paper tables |
| Orbach et al., IBM [25] | Chunk size, overlap, 3 embedding models, top-k, 3 answer models, 162 settings, 5 sets | Answer model, largest effect on most sets. Best against worst setting 0.11 to 0.26 answer correctness | Chunk size and overlap below 0.02, overlap often not significant |
| Wang et al. [24] | Chunk size, chunking technique, embedding, retrieval method, reranker, repacking, summarization | Retrieval method, 0.428 to 0.478 average score | Chunk size within 3.3 points of faithfulness from 128 to 1,024 tokens, -17.2 at 2,048 |
| Akarsu et al. [30] | 9 retrieval methods, 2 answer models | Reranker, +12.1 recall@5. Hybrid over dense, +10.8 | Document summary prefix +2.2 to +2.8. No chunking test |
| Dadopoulos et al. [29] | Hybrid search, 2 rerankers, metadata prefix, query rewriting, chunk expansion | Reranker, +8.5 F1 | Metadata prefix +3.5 to +7.2 |
| Mankour et al. [28] | Hybrid search, reranker, parser output, 7 answer models | Answer model, about 35% to 62% correct | Docling HTML -4.8. BM25 +4.3, reranker +3.0 |
| Finardi et al., Portuguese [27] | 6 retrievers, 2 chunking strategies, 4 answer models, number of chunks | Retriever, MRR@10 0.565 to 0.919 | Sentence window -0.27 of 10 |
| Singh [31] | 3 parsers, 3 chunkers, 5 embedding models, BM25 | Parser and chunker interaction, -0.077 recall@10 | Parser +0.039, chunker +0.028 |
| Qu et al. [2] | 3 chunker families, 3 embedding models | Embedding model, 7.44% | Chunker within about 3 points on real documents |
| Shaukat et al. [17] | 36 chunkers, 5 embedding models | Chunker, 21.5 points of nDCG@5 | The exception, with a 100-character baseline |
| Anthropic, vendor [1] | Context line, BM25 on extended text, reranker | Context line, 2.0 points of recall@20 | BM25 added 0.8 and the reranker 1.0 |
| Microsoft, vendor [22] | Keyword, vector, hybrid, reranker, chunk size, boundaries | Reranker on hybrid, +11.7 nDCG@3 | Chunk size 7.5, boundaries and overlap 1.5 to 3.0 points of recall@50 |

The pattern over these studies:

- The answer model, the retrieval method and a reranker each moved results by 8 to 49 points.
- Chunking method, overlap and chunk size inside a sensible range moved results by 0 to 6 points, with single results of 13 points for overlap.
- Parsing moved results by 5 to 20 points on tables, charts and scans, and by about 6 points of retrieval on digital PDFs.
- Gross mistakes moved results as much as the retrieval method: chunks of 64 or 2,048 tokens, a parser that garbles tables, or a transcript with a third of the words wrong.
- This agrees with the existing project research, which holds +4.3 points for syntax-tree chunks against 15 to 20 points for the retrieval method, routing and index size.

## 6. Measurement

The bake-off gives every arm the same ingestion output (ADR-0018), so it cannot show any of the effects above. The cheapest way to show them is a set of ingestion variants inside one fixed arm, scored on retrieval first.

**Arm.** The one-shot hybrid arm of the code retrieval research (BM25 index, embedding index and reranker, no agent) [53]. It has no sampling step, so one run per variant is enough.

**Variants.** Each is one extra index build over the test dataset.

| Variant | What changes | What it measures |
|---|---|---|
| I0 | Baseline as decided in ADR-0019 | Reference |
| I1 | Fixed 400-token windows over every source, no structure, no header | The whole value of type-aware chunking |
| I2, I3 | Prose chunk target 256 and 1,024 tokens, next to the baseline size | Chunk size |
| I4 | 20% overlap for prose, if the baseline has none, or none if it has | Overlap |
| I5 | Heading path, file path with enclosing symbol, or thread title in front of the search text, with no model | The free prefix |
| I6 | Model-written context line in the search text, written by one shortlisted model | The paid prefix |
| I7 | Context lines shuffled between chunks | Whether I6 gains from content or from added words, as in [12] |
| I8 | One record per issue or pull request thread, cut by tokens, instead of one per comment | Thread granularity, which no study measured |
| I9 | The synthetic workshop results rendered to PDF, Word and a scanned PDF and parsed by Docling, against chunks cut from their original Markdown | Parsing loss against a perfect parse, also in German |

**Metrics.**

- Recall@5 and recall@20 against the gold citations, per source type and per question language. Both need no judge.
- The share of gold citation ranges that lie inside one chunk. It is computed from the chunks alone and separates lost evidence from badly ranked evidence, as in [31].
- Chunk count and tokens per variant, for the cost side.
- Only for the baseline and the two variants with the largest retrieval gain: key-fact recall and citation precision and recall, with 3 repetitions as in the bake-off protocol. The answer model gets the same number of context tokens in every variant, not the same number of chunks, because larger chunks otherwise bring more text [12, 19].

**Number of runs.** Ten index builds with one retrieval run each, plus 9 answer runs (3 variants times 3 repetitions).

**Cost (estimate).** Embedding 5 million tokens ten times at 0.12 dollars per million is 6 dollars. The context lines for about 12,500 chunks cost 4 to 18 dollars with Haiku 5.5 or a Gemma 4 model. Reranking and the 9 answer runs cost under 5 dollars. The total stays under 30 dollars.

**What the runs can show.** With 50 gold questions the protocol detects paired differences of about 12 points, and 5 points need about 280 questions [53]. The effects in the literature are mostly 1 to 6 points. The runs can therefore show whether any ingestion choice has a large effect on this data, and the direction of the smaller ones, but not their size. Generated questions per chunk would add questions cheaply, but the synthetic test data research found that such questions favour wording overlap and inflate scores [53].

## Answers to the standing architecture questions

1. **Retrieval per source type.** The evidence backs a different cut per source type and no single method for all. Code is cut along the syntax tree, prose along headings with the heading path kept, tables are kept whole with title and header, transcripts are cut by turn. Table questions and narrative questions also favour different retrievers [14, 30, 32]. Nothing here changes how code and prose are searched.
2. **Scale.** Rule-based chunking costs seconds. A context line costs 3 to 73 dollars for 10,000 to 50,000 chunks and 11 to 31 dollars for a repository of 1 to 2 million lines (estimates). Propositions and questions per chunk multiply the number of records, by 6 in the Dense X index [4] and by 4 in index size for the German Civil Code [21].
3. **Knowledge organization.** The evidence supports routing by type to a parser and chunker at ingestion. A two-stage search that picks sections first lost 1.5 to 3.3 points [12], in line with the knowledge organization research [53].
4. **Embedding independence.** A context line is text, so it survives an embedding model change, and it helps BM25 more than it helps embeddings [51]. It becomes part of the embedded text, so the vector cache key of ADR-0020 must cover it, and an edit to one section re-embeds the file. Late chunking binds the index to one long-context embedding model and fell sharply with one of four models tested [10].
5. **Graphs.** Does not apply, except that a RAPTOR tree is a hierarchy built by a language model with a gain of 0.5 to 2.2 points [7]. Model-written pages are covered by #51.
6. **Location.** Today ingestion sends no content to any model (ADR-0018). Any model-written text at ingestion changes that, and the model policy then applies at ingestion. On AWS the candidates run in the EU. For a later on-device installation, one measurement gives 1.9 seconds per document on a data-centre GPU for context lines [11], and writing context lines for one statute of 2,455 sections took 38 to 52 minutes [21].
7. **Measurement.** Section 6. No bake-off arm tests ingestion today.

## Recommendations

Each recommendation names its evidence. None is a decision.

1. Keep rule-based, type-aware chunking with no language model as the default, as ADR-0019 decides. Evidence: structural boundaries gained 1.9 to 4.1 points over fixed windows [12], sections were best on German law [21], and semantic and model-based chunkers showed no consistent gain [2, 14, 18, 21].
2. Put the heading path, the file path with the enclosing symbol, and the thread title into the text that is embedded and indexed for BM25, for every chunk. Evidence: +0.8 to +1.0 points over section chunks and more than a model-written line [12], BM25 recall on tables 0.592 to 0.967 with titles [32]. ADR-0019 already does this for GitHub records.
3. Tune the chunk size per source type within about 250 to 1,000 tokens and treat overlap as a setting to test, not as a default. Evidence: [9, 13, 14, 22, 24, 25].
4. Keep each table whole with its title, header and caption, and check tables first when judging the parser. Evidence: [8, 15, 16, 32].
5. Measure Docling on the project's own pages, including German scans and tables, before relying on it. Evidence: 5 to 22 points of loss from text recognition [8], 20 points between parsers on tables [15], -4.8 points from unprocessed Docling HTML [28], and no German measurement anywhere.
6. Do not make a model-written context line part of the default ingestion. Test it as variant I6 next to the free prefix I5. Evidence: 0 to 3 points in independent tests on ordinary documents [10, 12, 30, 52], 20 points only where chunks are unreadable alone [11], and it changes ADR-0018 and ADR-0020.
7. If model-written text is ever stored, keep it in its own field that is searched but never shown as evidence and never cited. Evidence: section 3, "How answers still cite the source".
8. Do not add a second model that checks the first at ingestion. Where a model extracts facts that a human confirms anyway, such as decision log entries (ADR-0004), two models can sort the items for review. Evidence: agreed answers were 94% correct and disagreed ones 41% to 50% [39]. A second model misses the shared errors: 42% to 46% of each model's errors were in agreed answers [39], and two wrong models chose the same wrong answer 42% to 60% of the time [42]. A human check of model output matched human-only work [40].
9. Keep rules with go-enry and Magika for file classification. A language model check stays deferred until legacy test files exist, as ADR-0019 says. Evidence: no study found, and the pass would cost under 1 dollar when it is needed.
10. Add the runs I0 to I9 of section 6 to the bake-off plan.
11. Spend tuning effort first on hybrid search, the reranker and the answer model. Evidence: section 5.

## My own judgment

These are my views, not findings.

- The owner asked whether classification and chunking are the most critical part. On the evidence they are not, as long as the ingestion avoids gross errors. They set a ceiling, because what was never parsed or was cut apart cannot be retrieved. The current rules already avoid the large losses in the literature, which are chunks that are far too small or too large, tables turned into loose text, and text recognition that drops words.
- Parsing is the part of ingestion I would worry about most, because its losses are large on exactly the material clients send (scanned PDFs, tables, slides) and because no German number exists.
- I expect the context line to matter less here than in Anthropic's test. Code chunks carry their symbol, Markdown chunks their heading path and GitHub records their header already. Transcript turns are the one source where a chunk often says little alone, so that is where I would look first in run I6.
- An independent checker model for a classifying model is not worth building for V1. A wrong keep adds noise, a wrong skip is listed in the ingestion report, and the consultant can force-include a path. A sample check by a human of the report covers the same risk at lower cost.
- The 12-point detection limit of the gold set is the real constraint on section 6. I would run the variants anyway, because a run that shows "no effect above 12 points" already answers the owner's question for this data.

## Method and limits

- I read the ticket, the map, `GLOSSARY.md`, ADR-0017 to ADR-0020 and the research notes on the branches `research/github-ingestion`, `research/document-ingestion`, `research/knowledge-organization`, `research/code-retrieval`, `research/embedding-lock-in`, `research/aws-models`, `research/source-file-types`, `research/synthetic-test-data` and `research/local-evaluation`.
- I searched the web and read each cited source on 2026-11-20: arXiv abstract and HTML pages, vendor and practitioner pages, and two GitHub repositories. Abstracts of three medical papers came from the Europe PMC API. I ran no experiment and no code from other repositories.
- All eight items the owner recalled exist and say what this note reports: Anthropic's write-up [1], the semantic chunking paper [2], late chunking [3], Dense X Retrieval [4], doc2query [5], RAPTOR [7], OHRBench [8] and Chroma's report [9].
- Most numbers were read from the HTML version of a paper through a tool that summarises the page, not from the PDF. I checked figures against each other where a source gave them twice and noted the conflicts I saw. One number in Dense X is given as +9.3 in the text and follows as +9.7 from the table. This note uses the table. Numbers of El Bachyr et al. [14] come from my own text extraction of the PDF.
- Not read, because the server refused access: Snowflake's engineering post on chunking for financial filings, the hospital study with German documents [54], and the medRxiv version of [39]. A German practitioner article on chunking gave no numbers and is not used.
- Where a paper gives values only in figures, this note says so and gives no number: the German Civil Code results beyond the two best [21], the document mismatch rates in [37], and the middle noise levels in [33].
- Found nothing on: chunking of workshop minutes or transcripts, chunking of issue and pull request threads, splitting tables by rows, a language model for file classification, the error rate of model-written context lines, parsing errors on German documents, the effect of a 4% word error rate, and Docling's default pipeline measured end to end.
- Almost all evidence is English. Results on financial filings dominate the multi-factor studies. No study uses a corpus that mixes code, documents, issue threads and transcripts of one software project.
- The cost table rests on my token estimates and on list prices in the models research, which that note marks as needing confirmation for Claude Haiku 5.5.
- Several sources are preprints without review, some by a single author [31, 47] or informal [50, 52]. They are marked where used.

## Sources

Papers and preprints:

1. Anthropic. Introducing Contextual Retrieval. 2024-09-19. Vendor write-up. https://www.anthropic.com/news/contextual-retrieval
2. Qu, Tu, Bao. Is semantic chunking worth the computational cost? arXiv 2410.13070, 2024-10-16. Two authors at Vectara. https://arxiv.org/abs/2410.13070
3. Günther, Mohr, Williams, Wang, Xiao. Late chunking: contextual chunk embeddings using long-context embedding models. arXiv 2409.04701, v3 2025-07-07. Jina AI and Weaviate, vendor. https://arxiv.org/abs/2409.04701
4. Chen et al. Dense X Retrieval: what retrieval granularity should we use? EMNLP 2024. arXiv 2312.06648. https://arxiv.org/abs/2312.06648
5. Nogueira, Yang, Lin, Cho. Document expansion by query prediction. arXiv 1904.08375, 2019. https://arxiv.org/abs/1904.08375
6. Gospodinov, MacAvaney, Macdonald. Doc2Query--: when less is more. ECIR 2023. arXiv 2301.03266. https://arxiv.org/abs/2301.03266
7. Sarthi et al. RAPTOR: recursive abstractive processing for tree-organized retrieval. arXiv 2401.18059, 2024-01-31. https://arxiv.org/abs/2401.18059
8. Zhang et al. OCR hinders RAG: evaluating the cascading impact of OCR on retrieval-augmented generation (OHRBench). ICCV 2025. arXiv 2412.02592, v4 2025-08-30. https://arxiv.org/abs/2412.02592
9. Smith, Troynikov. Evaluating chunking strategies for retrieval. Chroma technical report, 2024-07-03. Practitioner report by a vendor. https://www.trychroma.com/research/evaluating-chunking
10. Merola, Singh. Reconstructing context: evaluating advanced chunking strategies for retrieval-augmented generation. Workshop at ECIR 2025. arXiv 2504.19754. https://arxiv.org/abs/2504.19754
11. Conti et al. Context is gold to find the gold passage: evaluating and training contextual document embeddings (ConTEB). arXiv 2505.24782, v2 2025-06-06. https://arxiv.org/abs/2505.24782
12. Kuehlkamp, Moreira, Rund. Does document structure help dense retrieval? A placebo-controlled ablation of four mechanisms across two corpora. arXiv 2610.10170, 2026-10-07. https://arxiv.org/abs/2610.10170
13. Bhat, Rudat, Spiekermann, Flores-Herr. Rethinking chunk size for long-document retrieval: a multi-dataset analysis. Fraunhofer IAIS. arXiv 2505.21700. https://arxiv.org/abs/2505.21700
14. El Bachyr et al. Empirical evaluation of PDF parsing and chunking for financial question answering with RAG. ICSE-SEIP 2026. arXiv 2604.12047. https://arxiv.org/abs/2604.12047
15. Hui et al. UDA: a benchmark suite for retrieval augmented generation in real-world document analysis. arXiv 2406.15187. https://arxiv.org/abs/2406.15187
16. Jimeno Yepes et al. Financial report chunking for effective retrieval augmented generation. Unstructured Technologies, vendor. arXiv 2402.05131. https://arxiv.org/abs/2402.05131
17. Shaukat, Adnan, Kuhn. A systematic investigation of document chunking strategies and embedding sensitivity. arXiv 2603.06976, 2026-03-07. https://arxiv.org/abs/2603.06976
18. Kreileder, Reisinger, Fischer. Evaluating chunking strategies for retrieval-augmented generation on academic texts. Deggendorf Institute of Technology. arXiv 2607.01852, 2026-07-02. https://arxiv.org/abs/2607.01852
19. Tencent Youtu Lab. HiChunk: evaluating and enhancing retrieval-augmented generation with hierarchical chunking. arXiv 2509.11552, v3 2025-10. https://arxiv.org/abs/2509.11552
20. Duarte et al. LumberChunker: long-form narrative document segmentation. arXiv 2406.17526, 2024-06. https://arxiv.org/abs/2406.17526
21. Prior, Milanova, Schultz. Chunking German legal code. Technical University of Munich. Workshop at ICAIL 2026. arXiv 2605.19806, v2 2026-05-29. https://arxiv.org/abs/2605.19806
22. Berntson. Azure AI Search: outperforming vector search with hybrid retrieval and reranking. Microsoft blog, 2023-09-18, updated 2024-11-22. Vendor. https://techcommunity.microsoft.com/t5/azure-ai-foundry-blog/azure-ai-search-outperforming-vector-search-with-hybrid/ba-p/3929167
23. Han. Finding the best chunking strategy for accurate AI responses. NVIDIA technical blog, 2025-06-18. Vendor. https://developer.nvidia.com/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/
24. Wang et al. Searching for best practices in retrieval-augmented generation. EMNLP 2024. arXiv 2407.01219. https://arxiv.org/abs/2407.01219
25. Orbach et al. An analysis of hyper-parameter optimization methods for retrieval augmented generation. IBM Research. arXiv 2505.03452, v3 2025-12-31. https://arxiv.org/abs/2505.03452
26. Li, Stenzel, Eickhoff, Bahrainian. Enhancing retrieval-augmented generation: a study of best practices. University of Tübingen. arXiv 2501.07391. https://arxiv.org/abs/2501.07391
27. Finardi et al. The chronicles of RAG: the retriever, the chunk and the generator. arXiv 2401.07883, 2024-01-15. https://arxiv.org/abs/2401.07883
28. Mankour et al. Climate Finance Bench. arXiv 2505.22752, 2025-05-28. https://arxiv.org/abs/2505.22752
29. Dadopoulos, Ladas, Moschidis, Negkakis. Metadata-driven retrieval-augmented generation for financial question answering. arXiv 2510.24402, 2025-10-28. https://arxiv.org/abs/2510.24402
30. Akarsu, Karaman, Mierbach. From BM25 to corrective RAG: benchmarking retrieval strategies for text-and-table documents. arXiv 2604.01733, 2026-04-02. https://arxiv.org/abs/2604.01733
31. Singh. Parser, chunking, and embedding interactions in retrieval-augmented generation over Indian government regulatory documents. Single author. arXiv 2609.31660, 2026-09-15. https://arxiv.org/abs/2609.31660
32. Ji, Glenn, Parameswaran, Hulsebos. TARGET: benchmarking table retrieval for generative tasks. arXiv 2505.11545, 2025-05-14. https://arxiv.org/abs/2505.11545
33. Tang, Zhang, Li, Zhou. Don't just listen, try planning: graph-based retrieval-generation agent for long-form audio meeting understanding. arXiv 2608.24048, 2026-08-25. https://arxiv.org/abs/2608.24048
34. Li, Wu, Liu, Lee. Spoken SQuAD: a study of mitigating the impact of speech recognition errors on listening comprehension. arXiv 1804.00320, 2018. https://arxiv.org/abs/1804.00320
35. Most et al. Lost in OCR translation? Vision-based approaches to robust document retrieval. Los Alamos National Laboratory. arXiv 2505.05666, 2025-05-08. https://arxiv.org/abs/2505.05666
36. Neeser et al. QuOTE: question-oriented text embeddings. arXiv 2502.10976, 2025-02-16. https://arxiv.org/abs/2502.10976
37. Reuter et al. Towards reliable retrieval in RAG systems for large legal datasets. arXiv 2510.06999, 2025-10-08. https://arxiv.org/abs/2510.06999
38. Weller et al. When do generative query and document expansions fail? A comprehensive study across methods, retrievers, and datasets. EACL 2024. arXiv 2309.08541. https://arxiv.org/abs/2309.08541
39. Khan et al. Collaborative large language models for automated data extraction in living systematic reviews. Journal of the American Medical Informatics Association 32(4), 638-647, 2025. https://doi.org/10.1093/jamia/ocae325
40. Gartlehner et al. Artificial intelligence-assisted data extraction with a large language model: a study within reviews. Annals of Internal Medicine 178, 1763-1771, 2025. https://doi.org/10.7326/ANNALS-25-00739
41. Buscemi, Hartling, Vandermeer, Tjosvold, Klassen. Single data extraction generated more errors than double data extraction in systematic reviews. Journal of Clinical Epidemiology 59, 697-703, 2006. https://doi.org/10.1016/j.jclinepi.2005.11.010
42. Kim, Garg, Peng, Garg. Correlated errors in large language models. ICML 2025. arXiv 2506.07962. https://arxiv.org/abs/2506.07962
43. Goel et al. Great models think alike and this undermines AI oversight. arXiv 2502.04313, v2 2025-06-12. https://arxiv.org/abs/2502.04313
44. Wang et al. Self-consistency improves chain of thought reasoning in language models. ICLR 2023. arXiv 2203.11171. https://arxiv.org/abs/2203.11171
45. Chen, Zaharia, Zou. FrugalGPT: how to use large language models while reducing cost and improving performance. arXiv 2305.05176, 2023-05-09. https://arxiv.org/abs/2305.05176
46. Ibrahim, Zaki. Evaluating decision models for text annotation in computational social science. arXiv 2609.24574, 2026-09-21. https://arxiv.org/abs/2609.24574
47. Qian. From unreliable to usable: decomposing failure modes and confidence signals in LLM entity extraction. Single author. arXiv 2610.20417, 2026-10-14. https://arxiv.org/abs/2610.20417
48. LLM-AggreFact leaderboard. https://llm-aggrefact.github.io/
49. Fratantonio et al. Magika: AI-powered content-type detection. Google. arXiv 2409.13768, 2024-09-18. https://arxiv.org/abs/2409.13768

Reproductions and informal tests:

50. Pantoja. contextual-retrieval-reproduction. Public repository that reruns Anthropic's benchmark with Anthropic's data. https://github.com/alexPantoja90210/contextual-retrieval-reproduction
51. Huang. Anthropic embeddings vs open-source rerank alternatives. DenserAI blog, 2024-10-07, updated 2026-02-07. Vendor of a retriever. https://denser.ai/blog/compare-open-source-paid-models-anthropic-dataset/
52. Tirth G. rag-chunking-benchmark. Public repository, 15 documents and 88 hand-written questions. https://github.com/Tirthg995/rag-chunking-benchmark

Project research:

53. Research notes of this repository on their `research/*` branches: [aws-models](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md), [knowledge-organization](https://github.com/Niko4417/OPT-cc/blob/research/knowledge-organization/docs/research/knowledge-organization.md), [github-ingestion](https://github.com/Niko4417/OPT-cc/blob/research/github-ingestion/docs/research/github-ingestion.md), [document-ingestion](https://github.com/Niko4417/OPT-cc/blob/research/document-ingestion/docs/research/document-ingestion.md), [code-retrieval](https://github.com/Niko4417/OPT-cc/blob/research/code-retrieval/docs/research/code-retrieval.md), [local-evaluation](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md), [synthetic-test-data](https://github.com/Niko4417/OPT-cc/blob/research/synthetic-test-data/docs/research/synthetic-test-data.md), [local-transcription](https://github.com/Niko4417/OPT-cc/blob/research/local-transcription/docs/research/local-transcription.md).

Found but not read:

54. Evaluation of chunking and embedding strategies for local document retrieval using an open-source LLM in a hospital. Studies in Health Technology and Informatics, 2025. The page returned HTTP 403. https://journals.sagepub.com/doi/10.3233/SHTI251383
