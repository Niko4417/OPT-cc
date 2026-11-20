# Evidence for generating a synthetic test dataset

Research for the test dataset of the project memory. It reviews the pipeline in the section "Generating workshop results" of [`test-dataset.md`](https://github.com/Niko4417/OPT-cc/blob/research/test-dataset/docs/research/test-dataset.md) against published evidence. The metric set and the bake-off protocol are in [`local-evaluation.md`](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md) and are not repeated here. Sources were read on 2026-11-20.

## Question

What does published evidence say about how to generate a synthetic test dataset for evaluating a retrieval-augmented question answering system? Which steps of the planned pipeline does that evidence support, contradict or leave untested?

## Short answer

- Synthetic question sets rank retrieval systems much like human question sets in three studies (Kendall tau 0.82, 0.84 and 0.93). The agreement fails in four measured cases: when the answer model differs between systems, when only the source passage counts as relevant (tau 0.157), when the synthetic questions are phrased unlike real ones, and for absolute scores.
- Model-written questions are longer, better formed, closer to the wording of the source and easier than human questions. Every study in section 1 that compared score levels found higher scores on synthetic questions. One study measured a drop of 29 to 37 points in Hit@5 between synthetic and authentic queries.
- Plan-first document generation with planted facts is the standard method in published benchmarks (RAGEval, LoCoMo, LongMemEval, HERB, DRBench). Fictional content is also the accepted defense against memorization (RepLiQA). The planned steps 1, 2 and 5 follow that method.
- The same benchmarks report that the generated text often fails to contain what the plan says. Human annotators edited 15% of dialog turns in LoCoMo and 70% of sessions in LongMemEval. The planned pipeline has no check that a planted fact is present in the output, and it reads only 3 of about 12 workshop results.
- A transcript written by one model in one pass scored 3 of 5 for naturalness against 4.5 for a multi-agent method and 4.5 for real meetings (FAME). No method in a second study reproduced the disfluency of real transcripts.
- No study tests whether two synthetic sets from one pipeline work as an independent tuning set and check set. Related evidence points toward using different writer models.
- I found no study of synthetic test data for German workshop minutes, and none that measures retrieval on synthetic against real meeting transcripts.

## Verdicts for the six planned steps

| Step | Verdict | Evidence |
| --- | --- | --- |
| 1. Fact sheet of real anchors, the model cites only those | Supported, indirectly | Facts-first generation is used by RAGEval, PhantomWiki and FAME. PhantomWiki dropped model rephrasing because it introduced hallucinated facts. No benchmark I found uses a manifest of real repository anchors, so the exact step is untested. |
| 2. Plan file per workshop with 2 to 4 planted facts | Supported | LoCoMo (event graphs), LongMemEval (evidence statements), DRBench (injected insights), HERB (workflows linked to queries) and RAGEval (configurations) all plan content before writing. DRBench and HERB also plan distractors, which the planned step lacks. |
| 3. One model writes minutes or transcript, fixed temperature and seed | Contradicted for single-pass transcripts. No evidence found for fixed temperature and seed. Mixed evidence on using the answer model as the writer model. | FAME rates a single-model transcript 3 of 5 for naturalness, against 4.5 for its multi-agent method. Devanathan et al. find no method that matches real transcripts on disfluency. Silencer measures a self-bias of 0.06 to 0.10 when a model is tested on its own benchmark. ProMQA and Rahmani et al. find none. |
| 4. Script checks references, checker model checks citing sentences, flagged files are regenerated | Supported, with one gap | RAGEval, MultiHop-RAG, UAEval4RAG and FAME all use a model-based check, and FAME regenerates scenes up to three times. The gap is the missing check that planted facts appear in the text. LoCoMo and LongMemEval needed human edits for exactly that failure. |
| 5. Gold questions from the plans, half German and half English, cross-language pairs | Supported for deriving questions from plans. Contradicted for relying on model-phrased questions only and on a single gold citation per fact. No evidence found for the 50/50 language split. | HERB, DRBench, PhantomWiki and LongMemEval derive questions and answers from the plan. Rahmani et al. measure tau 0.157 when only the seed passage counts as relevant. Fensore et al. measure BM25 MAP 0.557 for questions worded like the document and 0.201 for questions worded differently. XRAG shows cross-language questions are feasible and harder. |
| 6. A person reads every gold question and a sample of 3 workshops | Supported for reading every question. Contradicted by published practice for the sample of 3. | Human review rejected or corrected between 1% and 95% of generated questions in the benchmarks listed in section 6. LoCoMo verified all 50 conversations and LongMemEval edited about 70% of sessions. |

## 1. Validity

The question is whether a system ranking on synthetic questions matches the ranking on human-written questions.

| Study | Setup | Agreement |
| --- | --- | --- |
| Rahmani et al., SIGIR 2024 ([arXiv 2405.07767](https://arxiv.org/abs/2405.07767)) | TREC Deep Learning 2023, 51 real and 31 synthetic queries (13 from T5, 18 from GPT-4), 31 systems, NDCG@10 | Kendall tau 0.8151 between synthetic and real queries with human judgments. Tau 0.8568 with GPT-4 judgments. Tau 0.157 when only the passage the query was generated from counts as relevant. |
| Yin et al., DisastIR ([arXiv 2505.15856](https://arxiv.org/abs/2505.15856)) | 48 passages, each with one model-written and one expert-written query | Kendall tau 0.9264 between model rankings on NDCG@10 |
| van Elburg et al., SynDAiTE workshop at ECML-PKDD 2025 ([arXiv 2508.11758](https://arxiv.org/abs/2508.11758)) | Four datasets with about 100 human and 100 GPT-4o questions each, two of them proprietary | With retriever settings varied, mean tau was 0.84 with BLEU, 0.67 with ROUGE and semantic similarity, and 0.08 with context precision. With generator models varied, tau was "low or even negative" for several metric and dataset pairs. No significance test, because few systems were compared. |
| Hong et al., Chroma technical report 2025, a practitioner report ([link](https://www.trychroma.com/research/generative-benchmarking)) | Generated queries against production queries of the Weights and Biases support bot | Embedding model order mostly preserved. Recall@10 was higher on generated queries, at 0.530 against 0.439 for text-embedding-3-small and 0.602 against 0.552 for text-embedding-3-large. |
| Kucia and Gawlik, CIKM 2026 ([arXiv 2609.14579](https://arxiv.org/abs/2609.14579)) | University chatbot, 1,851 generated questions against 322 student queries, of which 165 could be scored | Hit@5 fell from 0.820 to 0.527 (dense, ChromaDB), from 0.849 to 0.545 (dense, Qdrant) and from 0.896 to 0.527 (hybrid). The hybrid setup led on synthetic questions and did not lead on authentic ones. |
| Hussain and Nielbo, preprint 2026 ([arXiv 2605.27220](https://arxiv.org/abs/2605.27220)) | Danish National Encyclopedia, 1,000 queries per condition, real queries from search logs | Synthetic queries suggested that over 90% of queries need query augmentation by a language model. On real queries 27.8% needed it. |

Outside retrieval, Maheshwari et al. ([arXiv 2409.11968](https://arxiv.org/abs/2409.11968)) report Spearman correlations of 0.94 for intent detection and as low as 0.20 for one named-entity setting. Yuan et al. (Silencer, NeurIPS 2025, [arXiv 2505.20738](https://arxiv.org/abs/2505.20738)) report an average Pearson correlation of 0.655 between model scores on a model-written benchmark and on a human-annotated benchmark, and 0.833 after their mitigations.

Conditions under which agreement breaks:

1. The systems differ in the answer model. van Elburg et al. name "stylistic bias favoring certain generators" as a possible cause.
2. Gold labels are sparse. Rahmani et al. write that the seed-passage-only condition "does not give similar results".
3. The synthetic questions are distributed unlike real ones. Kucia and Gawlik measured 15.7 words per synthetic question against 6.8 per authentic query, and 165 sources covered against 53.
4. Absolute scores are compared. Rahmani et al., CIKM 2025 ([arXiv 2506.10301](https://arxiv.org/abs/2506.10301)) fit a linear mixed-effects model and find that GPT-4-written queries raise NDCG by 0.059 (p < 0.001). They conclude the bias matters for absolute performance and less for comparing systems. Their data is a single test collection.

The bake-off in `local-evaluation.md` holds the answer model fixed and varies the stack, which is the condition where van Elburg et al. found agreement. The Kucia and Gawlik result still applies, because the stacks differ in retrieval architecture.

## 2. Known biases

**Lexical overlap.** Karpukhin et al. ([arXiv 2004.04906](https://arxiv.org/abs/2004.04906)) explain BM25 beating their dense retriever on SQuAD (68.8% against 63.2% top-20 accuracy) by the fact that "the annotators wrote questions after seeing the passage". Fensore et al. ([arXiv 2606.12789](https://arxiv.org/abs/2606.12789)) generated questions with DataMorgana and retrieved with BM25. Questions worded like the document reached MAP 0.557 (n = 517) and questions worded differently reached 0.201 (n = 483). Linguistic variation also separated answer quality more than question complexity did (range 0.077 against 0.010 in answer similarity).

**Bias toward single-hop fact questions.** De Lima et al. ([arXiv 2411.19710](https://arxiv.org/abs/2411.19710)) classified questions generated with one simple prompt and found that "95% of generated data falls into the fact_single label". Recall differed between the best and worst question type by 4.8% (Natural Questions) to 42% (NewsQA), and the best retrieval weighting differed per type.

**Difficulty.** Rahmani et al. 2024 report that all system types score higher on synthetic collections, which "suggest[s] that synthetic test collections tend to be easier". Chroma reports that naively generated queries "yielded higher retrieval metrics than the ground truth". Kucia and Gawlik measured the 29 to 37 point drop in section 1.

**Unrealistic phrasing.** GPT-4 queries averaged 10.72 words against 5.76 for real queries (Rahmani et al. 2024). Hussain and Nielbo found that 21.0% of real queries were well-formed sentences, against 65.3% and 97.5% for two synthetic conditions. In the Kucia and Gawlik study, 49% of the authentic queries had no static source page and could not be scored. Every synthetic question could.

**Self-preference.** The evidence is mixed.

- Yuan et al. measured an average self-bias of 0.103 (math) and 0.058 (language understanding) when a model is tested on a benchmark it wrote. Wrong labels caused 67% to 80% of it, question domain 11% to 18% and language style 9% to 15%.
- Maheshwari et al. measured bias factors between -0.095 (Llama-3-70B) and +0.061 (Mixtral-8x7B) across six models.
- Hasegawa et al. (ProMQA, NAACL 2025, [arXiv 2410.22211](https://arxiv.org/abs/2410.22211)) "did not find an indication that a model scores higher on its generated questions".
- Rahmani et al. 2024 found "little to no bias towards LLM-based systems" in retrieval.
- Li et al. ([arXiv 2502.01534](https://arxiv.org/abs/2502.01534)) measured a preference leakage score of 23.6% when the judge is the same model that generated a student model's training data, 8.9% for the same family and series, and 2.8% for the same family and a different series. Their setup fine-tunes student models, which differs from ours.
- Balog et al., SIGIR 2025 ([arXiv 2503.19092](https://arxiv.org/abs/2503.19092)) show a model judge scoring a model-based ranker at NDCG@10 0.947 where human judges gave 0.747. They found no evidence that model judges favor rewritten documents.

**Mitigations with measurements.**

| Mitigation | Source | Measured effect |
| --- | --- | --- |
| Question categories and user categories set in a configuration | Filice et al., DataMorgana ([arXiv 2501.12789](https://arxiv.org/abs/2501.12789)) | 1,248 distinct part-of-speech templates against 573 for a plain prompt. Question categories carried the effect. The user categories were "marginal". |
| Steering generation with context and example queries | Chroma | KL divergence between the query-document similarity distributions of generated and real queries was 0.159, against 0.207 for naive generation |
| Rejecting questions that copy the source (ROUGE-L F1 above 0.35 or a shared 4-gram), then paraphrasing | Kuehlkamp et al. ([arXiv 2610.10170](https://arxiv.org/abs/2610.10170)) | 1,945 of 2,306 questions passed. Shipped questions had ROUGE-L 0.02 to 0.13 against their evidence. No before and after retrieval comparison is reported. |
| Paraphrasing by another model | Yuan et al. | Small. Self-bias fell from 0.1205 to 0.1074 on math. |
| Several writer models | Yuan et al., Maheshwari et al. | Self-bias fell from 0.1205 to 0.0218 on math with an unweighted ensemble. Maheshwari et al. recommend averaging over several writer models. |
| Expert filtering of generated queries | Rahmani et al. 2024 | Assessors kept 31 of 97 generated queries. |
| Round-trip filter | Saad-Falcon et al., ARES ([arXiv 2311.09476](https://arxiv.org/abs/2311.09476)) | ARES keeps a query only if a retriever returns its source passage first. No ablation was reported in what I read. |
| A human-written subset | Kucia and Gawlik | They recommend collecting 50 to 100 authentic queries and comparing them with the synthetic set. Their own comparison is the measurement. |
| Unanswerable questions | Peng et al., UAEval4RAG ([arXiv 2412.12300](https://arxiv.org/abs/2412.12300)) | 92% of generated unanswerable questions matched their category in a human check of 100 per dataset. |

Published shares of unanswerable or false-premise questions are 6% (LongMemEval), 11.8% (MultiHop-RAG), 12% (CRAG), about 20% (RepLiQA) and 46% (HERB).

## 3. Generating the documents

**Plan-first and schema-first generation.**

- RAGEval ([arXiv 2408.01262](https://arxiv.org/abs/2408.01262)) goes from a schema to a configuration of concrete values, then to a document, then to questions, references and key points. Human raters ranked its documents first "in over 85% of the cases" against zero-shot and one-shot generation. Eight annotators scored 420 question, reference and answer samples between 4.76 and 4.94 of 5.
- LoCoMo ([arXiv 2402.17753](https://arxiv.org/abs/2402.17753)) gives each agent a persona and a graph of up to 25 dated events, then lets two agents talk. Annotators "edited nearly 15% of the dialog turns" and removed events that did not appear in the conversation.
- LongMemEval ([arXiv 2410.10813](https://arxiv.org/abs/2410.10813)) writes the question first, splits the answer into evidence statements and plants each statement in a simulated chat session. "Roughly 70% of the sessions are human edited."
- HERB ([arXiv 2506.23139](https://arxiv.org/abs/2506.23139)) is the closest published analog to our corpus. It holds 39,190 synthetic items of fictional software organizations, among them 321 meeting transcripts, 3,562 pull requests and 33,632 chat messages. Experts wrote 41 query templates first. Nine workflows then generated the documents that answer them, plus distractors. It has 815 answerable and 699 unanswerable questions. GPT-4o wrote all of it, and the paper reports no full human audit.
- DRBench ([arXiv 2510.00172](https://arxiv.org/abs/2510.00172)) generates insights and distractor insights, then inserts each into a file and fills the rest with unrelated content. Ground truth is known by construction. Annotators approved 96% of questions (72 of 75 votes).
- PhantomWiki ([arXiv 2502.20377](https://arxiv.org/abs/2502.20377)) generates facts first and renders articles from templates without a language model, because model rephrasing hallucinated facts.

**Fictional content against memorization.** RepLiQA ([arXiv 2406.11811](https://arxiv.org/abs/2406.11811)) paid writers for 3,591 fictional documents per split so that no model has seen them. GPT-4o reached 86.80 recall on TriviaQA and 60.85 on RepLiQA. Without the document it reached 8.59. Planted facts that exist only in a synthetic document follow the same principle. I found no paper that plants fictional facts into a public code repository.

**Memorization of public repositories.** Liang et al. ([arXiv 2506.12286](https://arxiv.org/abs/2506.12286)) asked models to name the buggy file from the issue text alone. Accuracy reached 76% on SWE-Bench Verified and at most 53% on repositories outside it. I found no measurement for `thunderbird/thunderbird-android` or `shopware/shopware`.

**Closed-book baselines.** Benchmarks measure memorization by running the answer model without retrieval. The results are 34% accuracy for the best model on CRAG ([arXiv 2406.04744](https://arxiv.org/abs/2406.04744)), 6.30% for GPT-4o on XRAG ([arXiv 2505.10089](https://arxiv.org/abs/2505.10089)) and 0% on NQ-UTD in Cocktail ([arXiv 2405.16546](https://arxiv.org/abs/2405.16546)). MultiHop-RAG ([arXiv 2401.15391](https://arxiv.org/abs/2401.15391)) and XRAG pick news published after the model cutoff for the same reason.

## 4. Synthetic meeting minutes and transcripts

Kirstein et al. (FAME, [arXiv 2502.13001](https://arxiv.org/abs/2502.13001), listed in the ACL Anthology as 2025.findings-acl.599) generated 500 English and 300 German meetings from Wikipedia articles. Each speaker is a separate agent that sees only its own profile and source snippets. Six annotators rated 30 meetings per corpus.

| Corpus | Naturalness (median, 1 to 5) | Words per meeting | Turns per meeting |
| --- | --- | --- | --- |
| FAME English | 4.5 | 6,223 | 405 |
| FAME German | 4 | 6,272 | 393 |
| One model, one pass, full knowledge | 3 | not reported | not reported |
| AMI (real) | 4.5 | 4,938 | 514 |
| ICSI (real) | 4.5 | 9,889 | 758 |
| QMSum (real) | 4.5 | 7,303 | 521 |

Further findings from FAME:

- The single-pass baseline produced "shallow, repetitive content".
- Annotators found synthetic cues, such as repeated phrases and overly polite turns, in 57 of 800 meetings.
- FAME scored 2 for speaker dynamics against 3 for QMSum, and 4 for information density against 2.5.
- With mid-sized open models, 2 of 25 transcripts needed major edits, against 1 of 25 with GPT.
- The multi-agent method cost about 30 minutes per meeting, against about 3 minutes for one pass.
- The planned length of 3,000 to 6,000 words per transcript is in the range of AMI and below ICSI and QMSum.

Devanathan et al. ([arXiv 2508.18210](https://arxiv.org/abs/2508.18210)) compared six generation methods with real contact center transcripts on 18 metrics in English, Spanish, French and Canadian French. No method matched the real distribution of disfluency, turn-level sentiment or language complexity in any language. They conclude that "Disfluency and ASR Noise remain difficult to replicate". These are calls, not meetings.

Two findings bear on what matters for retrieval and answering.

- Thonet et al. (ELITR-Bench, [arXiv 2403.20262](https://arxiv.org/abs/2403.20262)) wrote 271 questions by hand over 18 real meeting transcripts from speech recognition. When they raised the word error rate from 21% to 71%, the gap between models widened. A clean synthetic transcript therefore hides differences that real transcripts expose.
- Dai et al., KDD 2024 ([arXiv 2310.20501](https://arxiv.org/abs/2310.20501)) show that neural retrievers rank model-written text above human text of the same meaning (section 5).

Golany et al. (MISeD, [arXiv 2405.01121](https://arxiv.org/abs/2405.01121)) generated questions over 225 real QMSum meetings with a model. Annotators removed 6% of the queries and corrected 11% of the responses.

**German and multilingual.**

- FAME German lags FAME English by 0.5 points in naturalness.
- XRAG verified questions in English and had professionals translate them into German, Spanish, Chinese and Arabic. GPT-4o reached 57.58% with mixed-language documents and 61.16% when the supporting articles were switched to English.
- ARES judges trained on synthetic data fell to tau 0.33 when the language changed.
- The Avemio German RAG benchmark ([dataset card](https://huggingface.co/datasets/avemio/German-RAG-LLM-HARD-BENCHMARK)) contains German synthetic meeting conversations written by Llama-3.1-70B. Its card reports checks by language models only and no human validation.

I found no study of synthetic meeting minutes as opposed to transcripts. I found no study that compares retrieval results on synthetic and real meeting transcripts. I did not find a real German meeting corpus among the corpora named in the task (AMI, ICSI, QMSum and ELITR are English, and ELITR also has Czech).

## 5. Mixing real and synthetic content in one corpus

- LongMemEval builds each chat history from "25% ShareGPT, 25% UltraChat, and 50% simulated sessions" and plants the evidence sessions among them.
- DRBench combines public web pages with generated internal files that carry the planted insights.
- Cocktail adds a model-rewritten copy of every document to 16 retrieval datasets. Neural retrievers preferred the rewritten copies on nearly all datasets, with an average relative difference beyond 25%. Better retrievers showed more of this bias (Pearson -0.772). Adding a cross-encoder reranker to BM25 moved the relative difference from +22.0 (favoring human text) to -32.9 (favoring model text).
- Dai et al. report the same for single models. On SciFact, NDCG@1 moved 25.5% to 47.0% toward the model-written copy for four neural retrievers, and BM25 moved 23.9% toward the human text. They explain it with lower perplexity and less noise in model-written text.
- Tan et al., ACL 2024 ([arXiv 2401.11911](https://arxiv.org/abs/2401.11911)) find that GPT-4, GPT-3.5 and Llama 2 favor generated context over retrieved context when the two conflict, even when the generated one is wrong.
- Ouyang et al. (HoH, [arXiv 2503.04800](https://arxiv.org/abs/2503.04800)) built pairs of current and outdated statements from real Wikipedia revisions. One outdated passage next to a current one cut the overall score of Llama-3.1-70B by more than 24%. This supports testing superseded decision records. It does not use synthetic documents.

What follows for our corpus is an inference, not a measured result. The synthetic workshop results sit next to human-written repository content. A stack that leans on dense retrieval and reranking may find planted facts more easily than facts in real documents, and a stack that leans on BM25 may do the opposite. The source bias studies compared rewrites of the same content, which is a different setup from ours.

## 6. Human review

| Benchmark | What people checked | Share rejected or changed |
| --- | --- | --- |
| Rahmani et al. 2024 | 97 generated queries, by professional assessors | 68% rejected (31 kept) |
| LongMemEval | About 1,000 generated questions per type, all sessions | About 95% of questions rejected or rewritten ("final yield rate is about 5%"). About 70% of sessions edited. |
| XRAG | 2,950 generated cross-document questions | 90% natural, 72% answerable, 54% with a correct answer after corrections. About 1,500 kept. |
| ProMQA | 500 generated questions | 20% rejected (401 kept) |
| LoCoMo | All 50 conversations | Nearly 15% of turns edited, 19% of images removed or replaced |
| MISeD | All 443 dialogs | 6% of queries removed, 11% of responses corrected |
| RepLiQA | Human-written documents and questions, vendor review | 5% to 10% rejected at first, falling later |
| CRAG | All questions and answers, two reviewers each | 5% of answers corrected, a further 2% updated |
| DRBench | First 15 tasks, 5 annotators | 4% of votes against (72 of 75 approved) |
| Fensore et al. | 110 questions, 2 annotators | 1.8% of answers unacceptable, 9.1% not free of hallucination. Agreement on answer correctness was Fleiss kappa 0.47. |
| HoH | 500 questions | 0.8% wrong answers |
| RAGEval | 420 samples, 8 annotators | About 5% or less scored below 4 of 5 |
| MultiHop-RAG | "a subset sample" of queries, plus a GPT-4 check of all | Not reported |
| HERB | 50 failed questions per model | No audit of the ground truth reported |

The share depends on what reviewers judged. Checks of answer correctness on single-document questions rejected 1% to 20%. Cross-document and multi-session questions lost 46% to 95%. Rahmani et al. rejected 68% of single-passage queries because they "did not look reasonable" or had too few or too many relevant documents.

Sample sizes that sources recommend or used for the human-labeled part:

- ARES needs "roughly 150-300 datapoints" of human labels. Below 100 to 150 it "cannot meaningfully distinguish" systems.
- Husain and Shankar, a practitioner guide ([link](https://hamel.dev/blog/posts/evals-faq/what-is-the-best-approach-for-generating-synthetic-data.html)), recommend writing 20 dimension tuples by hand, generating about 100 diverse traces and annotating at least 30 by hand. They state that "synthetic data cannot tell you how common a failure is in production".
- Kucia and Gawlik recommend 50 to 100 authentic queries.

## 7. Independence between the tuning set and the check set

I found no study that tunes a retrieval or question answering system on one synthetic set and checks it on a second set from the same pipeline. The related evidence is listed below.

- Geva et al. ([arXiv 1908.07898](https://arxiv.org/abs/1908.07898)) show for human-written datasets that models learn the annotator. Accuracy fell by up to 23 points on OpenBookQA, 10 on CommonsenseQA and 5 on MNLI for annotators absent from the training data. They conclude that "test set annotators should be disjoint from training set annotators". A writer model with fixed templates is the analog of a single annotator. This is an analogy, because we tune a pipeline and do not train a model.
- Li et al. show that the relation between the writer model and the judge matters, and that it weakens with distance (23.6% same model, 2.8% same family with a different series).
- Yuan et al. and Maheshwari et al. find that benchmarks averaged over several writer models track human benchmarks better than a benchmark from one writer model.
- Filice et al. find that the question categories in the template drive diversity (1,248 against 573 templates). Two sets from the same categories share the same phrasing patterns.
- Kucia and Gawlik and van Elburg et al. both name the single writer model as a limit of their own studies.

I found no evidence on seeds. A different seed changes the sample and keeps the writer model, the templates and the plan structure, which are the shared factors the sources above point to.

The second project differs from the first in its real content (repository, issues, decision records). That part of the check is independent whatever the synthetic pipeline does.

## 8. Size

No source I found states a minimum number of questions per question type.

- The validity studies reached stable rankings with small sets: 31 synthetic queries for 31 systems (tau 0.8151), 48 query pairs (tau 0.9264) and about 100 questions per dataset (van Elburg et al.). These rank systems that differ widely. van Elburg et al. ran no significance test.
- Polo et al., ICML 2024 ([arXiv 2402.14992](https://arxiv.org/abs/2402.14992)) find 100 curated examples enough to estimate a model's score on the 14,000-question MMLU. That concerns one model's score, not a difference between close systems.
- Published per-type counts are far above ours. The smallest types hold 180 questions in CRAG, 301 in MultiHop-RAG and 30 in LongMemEval (abstention, of 500). Fensore et al. used 400 questions per dimension.
- Thonet et al. ask for caution about a result on a type with 8 questions.
- Voorhees and Buckley, SIGIR 2002 ([NIST record](https://www.nist.gov/node/707561)) computed error rates for topic sets up to 25 and extrapolated beyond. Their abstract says the error rates "are larger than anticipated", "especially if few topics are used". I could not read the full paper, so I give no threshold from it.

`local-evaluation.md` already derives that 50 questions detect a paired difference of about 12 points. By the same formula and the same assumed standard deviation of 0.3, a type with 10 questions detects about 27 points and a type with 25 questions about 17 points. These are estimates. Per-type results at those sizes describe behavior and cannot rank stacks.

## Revised pipeline

Steps keep their numbers. "Unchanged" means the evidence gives no reason to change the step.

1. **Fact sheet.** Unchanged.
2. **Plan.** Unchanged, with two additions. The plan lists distractor statements that resemble a planted fact and differ in one detail, as DRBench and HERB do. The plan lists every document that states each fact, so that gold citations are complete (Rahmani et al., tau 0.157 with seed-only labels).
3. **Write.** Minutes are written in one pass, unchanged. Transcripts are written scene by scene with one call per speaker, where each speaker sees only its own brief (FAME, 4.5 against 3). The generated files are committed and tagged, so the dataset does not depend on reproducing a run.
4. **Check.** The reference check and the checker model stay. Two checks are added.
   - A script and the checker model confirm that every planted fact and every planned anchor appears in the text and that no distractor replaced it (LoCoMo, LongMemEval).
   - For each gold question, the answer model receives only the gold passages and must return the key facts. A question that fails is fixed or dropped. XRAG and HERB report this oracle setting (75.40% and 85.76%), and ARES filters by a round trip.
5. **Gold questions.** Questions still come from the plans. Five controls are added.
   - A fixed count per question type, with must-refuse questions at 10% to 20% of the set. Published benchmarks use 6% to 46%.
   - Phrasing categories in the generation prompt, in the DataMorgana manner. They include short keyword-style questions and questions worded unlike the source.
   - A filter that rejects a question with ROUGE-L F1 above 0.35 against its gold passage or a shared 4-gram (Kuehlkamp et al.). The share of similar and distant wording is recorded, because it moves BM25 results by a factor of 2.8 (Fensore et al.).
   - A human-written subset of at least 50 questions, written by consultants who have seen the workshop agenda and not the generated documents (Kucia and Gawlik; Karpukhin et al. on questions written after reading the passage).
   - A closed-book run of the answer model on every question. A question answered correctly without retrieval is marked as memorized and reported separately (RepLiQA, CRAG, XRAG).
6. **Review.** A person reads every gold question, unchanged. A person also reads every workshop result, not a sample of 3. With about 12 documents, a defect in one unread document is found with a probability of 25% when 3 are read. The review records how many questions and documents were rejected or changed.
7. **Reporting, new.** Results are reported separately for questions answered by synthetic and by real documents, for generated and human-written questions, and for German, English and cross-language questions. Absolute scores on generated questions are treated as an upper bound. When the first questions from real use exist, the rank agreement between the question sets is computed (Kendall tau, as in the studies of section 1).

### Changes that are my own judgment

The evidence for these is indirect, by analogy or absent.

- Use a writer model from another family than the answer model and the judge. The direct evidence on self-preference is mixed (section 2), and the bake-off holds the answer model fixed across stacks, so a bias would shift all stacks alike.
- Use a different writer model and reworded templates for the second project. This rests on the analogy to Geva et al. and on Li et al. and Yuan et al. No study tested it for retrieval evaluation.
- Do not use the evaluation judge (`gemma-4-31b-it`) as the checker model in step 4, or accept that the dataset then contains only citing sentences this judge can verify. I found no study of this.
- Make transcripts resemble speech recognition output. Options are to synthesize speech from the transcript and run it through the project's own transcription, or to add a few real meeting transcripts from QMSum as a control. ELITR-Bench and Devanathan et al. show the gap. Neither tested a fix for retrieval.
- Allow small hand edits of generated files and log them, in place of regenerating every flagged file. LoCoMo and LongMemEval edited by hand. FAME regenerated. No source compares the two.
- Have a second person review 20 to 30 gold questions and report the agreement. CRAG used two reviewers per item. Fensore et al. measured kappa 0.47 between two annotators.
- Hold at least 20 questions per type and present per-type results as descriptive.
- Write German questions in German instead of translating them. XRAG translated with professionals. FAME generated directly in German. No source compares the two for test questions.

## Method and limits

- I searched the web on 2026-11-20 and read arXiv HTML pages, one technical report, two dataset cards, the Ragas documentation and source file, one NIST record and one practitioner guide. A fetch tool extracted the facts from each page with a small language model. I did not read the PDFs line by line. I re-read three pages with a request for exact quotes (Rahmani et al. 2024, LongMemEval, XRAG) and the numbers held.
- All seven names from the task exist. RAGEval, ARES, RepLiQA, DataMorgana and MultiHop-RAG say what this note reports. CRAG is not model-written. Its questions come from over 600 templates over knowledge graphs and from human annotators. The Ragas documentation for test set generation ([link](https://docs.ragas.io/en/stable/concepts/test_data_generation/rag/)) describes a knowledge graph, query synthesizers for single-hop and multi-hop questions and scenarios with length, style and persona. It cites no validation against human-written questions and no quality numbers. The default distribution in the source ([file](https://github.com/explodinggradients/ragas/blob/main/src/ragas/testset/synthesizers/__init__.py)) gives equal weight to the single-hop specific, multi-hop abstract and multi-hop specific synthesizers.
- Not verified: the thresholds in Voorhees and Buckley 2002, the per-dataset tau values of the generator experiment in van Elburg et al. (they appear only in a figure), and the peer-reviewed venue of papers for which this note gives only an arXiv number.
- Several sources contradict themselves in details. FAME gives both "1 in 25" and "1 out of 30" transcripts with major edits. RAGEval gives two different document counts. The filter table in Guinet et al. ([arXiv 2405.13622](https://arxiv.org/abs/2405.13622)) does not add up, so this note does not use it.
- Most validity studies cover English web or encyclopedia text and compare retrievers. None covers a corpus of code, issues, decision records and workshop results, and none covers German.
- Three sources are preprints from 2026 without peer review that I could confirm (Hussain and Nielbo, Fensore et al., Kuehlkamp et al.). Kucia and Gawlik rests on 322 queries from 89 students and one writer model.
- The SIGIR 2025 LiveRAG report ([arXiv 2507.04942](https://arxiv.org/abs/2507.04942)) ran a challenge on 500 DataMorgana questions. It validates the judge against people (Pearson 0.88) and reports no check of the questions, so it does not bear on validity.
- I ran no code and no experiment. Every number is from the cited source, except the calculations marked as estimates.

## Sources

Validity and bias of synthetic questions:

- Rahmani et al., "Synthetic Test Collections for Retrieval Evaluation", SIGIR 2024. https://arxiv.org/abs/2405.07767
- Rahmani et al., "Towards Understanding Bias in Synthetic Data for Evaluation", CIKM 2025. https://arxiv.org/abs/2506.10301
- van Elburg et al., "Can we Evaluate RAGs with Synthetic Data?", SynDAiTE workshop at ECML-PKDD 2025. https://arxiv.org/abs/2508.11758
- Yin et al., "DisastIR". https://arxiv.org/abs/2505.15856
- Kucia and Gawlik, "Beyond Benchmark Scores: How Synthetic and Authentic Query Distributions Diverge in RAG Evaluation", CIKM 2026. https://arxiv.org/abs/2609.14579
- Hussain and Nielbo, "The Coverage Illusion". https://arxiv.org/abs/2605.27220
- Hong et al., "Generative Benchmarking", Chroma technical report, 2025 (practitioner report). https://www.trychroma.com/research/generative-benchmarking
- Husain and Shankar, "What is the best approach for generating synthetic data?", 2025 (practitioner guide). https://hamel.dev/blog/posts/evals-faq/what-is-the-best-approach-for-generating-synthetic-data.html
- de Lima et al., "Know Your RAG". https://arxiv.org/abs/2411.19710
- Filice et al., "Generating Diverse Q&A Benchmarks for RAG Evaluation with DataMorgana". https://arxiv.org/abs/2501.12789
- Carmel et al., "SIGIR 2025 LiveRAG Challenge Report". https://arxiv.org/abs/2507.04942
- Fensore et al., "How Fine-Grained Should a RAG Benchmark Be?". https://arxiv.org/abs/2606.12789
- Kuehlkamp et al., "Does Document Structure Help Dense Retrieval?". https://arxiv.org/abs/2610.10170
- Karpukhin et al., "Dense Passage Retrieval for Open-Domain Question Answering". https://arxiv.org/abs/2004.04906
- Saad-Falcon et al., "ARES", NAACL 2024. https://arxiv.org/abs/2311.09476
- Peng et al., "UAEval4RAG". https://arxiv.org/abs/2412.12300
- Ragas test set generation. https://docs.ragas.io/en/stable/concepts/test_data_generation/rag/

Self-preference and independence:

- Yuan et al., "Silencer", NeurIPS 2025. https://arxiv.org/abs/2505.20738
- Maheshwari et al., "Efficacy of Synthetic Data as a Benchmark". https://arxiv.org/abs/2409.11968
- Li et al., "Preference Leakage". https://arxiv.org/abs/2502.01534
- Hasegawa et al., "ProMQA", NAACL 2025. https://arxiv.org/abs/2410.22211
- Balog et al., "Rankers, Judges, and Assistants", SIGIR 2025. https://arxiv.org/abs/2503.19092
- Geva et al., "Are We Modeling the Task or the Annotator?". https://arxiv.org/abs/1908.07898

Generated documents, planted facts and memorization:

- Zhu et al., "RAGEval". https://arxiv.org/abs/2408.01262
- Maharana et al., "Evaluating Very Long-Term Conversational Memory of LLM Agents" (LoCoMo). https://arxiv.org/abs/2402.17753
- Wu et al., "LongMemEval". https://arxiv.org/abs/2410.10813
- Choubey et al., "Benchmarking Deep Search over Heterogeneous Enterprise Data" (HERB). https://arxiv.org/abs/2506.23139
- Abaskohi et al., "DRBench". https://arxiv.org/abs/2510.00172
- Gong et al., "PhantomWiki". https://arxiv.org/abs/2502.20377
- Monteiro et al., "RepLiQA". https://arxiv.org/abs/2406.11811
- Liang et al., "The SWE-Bench Illusion". https://arxiv.org/abs/2506.12286
- Yang et al., "CRAG", NeurIPS 2024 Datasets and Benchmarks Track. https://arxiv.org/abs/2406.04744
- Tang and Yang, "MultiHop-RAG". https://arxiv.org/abs/2401.15391
- Liu et al., "XRAG: Cross-lingual Retrieval-Augmented Generation". https://arxiv.org/abs/2505.10089
- Ouyang et al., "HoH". https://arxiv.org/abs/2503.04800
- Guinet et al., "Automated Evaluation of Retrieval-Augmented Language Models with Task-Specific Exam Generation". https://arxiv.org/abs/2405.13622

Meetings and transcripts:

- Kirstein et al., "You need to MIMIC to get FAME". https://arxiv.org/abs/2502.13001
- Devanathan et al., "Why Synthetic Isn't Real Yet". https://arxiv.org/abs/2508.18210
- Thonet et al., "ELITR-Bench". https://arxiv.org/abs/2403.20262
- Golany et al., "Efficient Data Generation for Source-grounded Information-seeking Dialogs" (MISeD). https://arxiv.org/abs/2405.01121
- Avemio, German-RAG-LLM-HARD-BENCHMARK dataset card. https://huggingface.co/datasets/avemio/German-RAG-LLM-HARD-BENCHMARK

Mixed corpora:

- Dai et al., "Neural Retrievers are Biased Towards LLM-Generated Content", KDD 2024. https://arxiv.org/abs/2310.20501
- Dai et al., "Cocktail". https://arxiv.org/abs/2405.16546
- Tan et al., "Blinded by Generated Contexts", ACL 2024. https://arxiv.org/abs/2401.11911

Size:

- Polo et al., "tinyBenchmarks", ICML 2024. https://arxiv.org/abs/2402.14992
- Voorhees and Buckley, "The Effect of Topic Set Size on Retrieval Experiment Error", SIGIR 2002. https://www.nist.gov/node/707561
