# Local evaluation of a project memory

Research for [issue #7](https://github.com/Niko4417/OPT-cc/issues/7), map [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). Sources were read on 2026-11-17. Keiko links point at the public repo [oscharko-dev/Keiko](https://github.com/oscharko-dev/Keiko) (Apache-2.0).

## Question

How can the answer quality of a project memory be measured without any cloud call? Can Ragas, DeepEval or TruLens use a local judge model behind LiteLLM? Which metrics need a judge model and which do not? How reliable are judges of the size of the lab pair (`mlx-community/Qwen3.6-35B-A3B-4bit` and `mlx-community/gemma-4-31b-it-4bit`)? How long does one evaluation run of about 50 gold questions take on a 64 GB Apple Silicon Mac?

A scope extension from the user adds three parts. Survey RAG evaluation methods and benchmarks. Pick a concrete metric set. Describe a fair protocol for running 2 or 3 candidate stacks head to head on the same gold set and the same models.

## Short answer

- Most of the signal needs no judge. Retrieval recall at k and MRR against gold citations, citation membership, citation precision and recall against gold citations, refusal on unanswerable questions, key-fact recall and latency are deterministic. They run in seconds and cannot drift with the judge.
- A judge is needed only for faithfulness and citation support (is a claim entailed by the text it cites) and for semantic answer correctness against a reference answer. All three frameworks accept an OpenAI-compatible LiteLLM URL. Ragas fits those metrics best.
- Use `gemma-4-31b-it-4bit` as the judge with thinking off, and keep `Qwen3.6-35B-A3B-4bit` as the answer model. A judge from another model family avoids self-preference.
- No published judge-agreement number exists for either lab model. The nearest evidence puts a 32B open model at about 77% balanced accuracy against German human faithfulness labels. That is enough to compare stacks over 50 questions, but not to trust one verdict on one answer. Calibrate the judge once against hand labels.
- One run of 50 questions takes about 7 minutes for answers and deterministic metrics, plus 17 minutes to 2.2 hours for the judge pass. A bake-off of 3 stacks with 3 repetitions each takes about 10 hours, one night.
- With 50 questions, a paired comparison detects differences of about 12 points or more. Smaller gaps need 100 to 300 questions.
- Ragas and DeepEval send usage telemetry by default. A fully local run sets `RAGAS_DO_NOT_TRACK=true` and `DEEPEVAL_TELEMETRY_OPT_OUT=1`.

## 1. Evaluation methods and benchmarks in the literature

### Frameworks and methods

| Work | What it measures | How | Relevance here |
| --- | --- | --- | --- |
| Ragas, Es et al. ([arXiv 2309.15217](https://arxiv.org/html/2309.15217v2)) | Faithfulness, answer relevance, context relevance without human references | LLM prompts. Agreement with human annotators on WikiEval: 0.95 faithfulness, 0.78 answer relevance, 0.70 context relevance, with gpt-3.5-turbo-16k (Table 1) | Library used for judge metrics (section 3) |
| ARES, Saad-Falcon et al., NAACL 2024 ([arXiv 2311.09476](https://arxiv.org/abs/2311.09476)) | Context relevance, answer faithfulness, answer relevance | Fine-tunes small judges on synthetic data, then corrects their error with prediction-powered inference over "a few hundred human annotations" | The correction idea applies to our calibration step. Fine-tuning judges is out of scope |
| RAGChecker, Ru et al., NeurIPS 2024 Datasets and Benchmarks ([arXiv 2408.08067](https://arxiv.org/abs/2408.08067), [repo](https://github.com/amazon-science/RAGChecker), Apache-2.0) | Overall precision and recall; retriever claim recall and context precision; generator context utilization, noise sensitivity, hallucination, self-knowledge, faithfulness | Claim extraction plus claim-level entailment through RefChecker, which calls any LiteLLM model with an `openai/` prefix and `api_base` ([RefChecker README](https://github.com/amazon-science/RefChecker)) | Best diagnostic split of retriever and generator errors. The repo was last pushed 2024-12-13. Its examples use Llama 3.1 70B as extractor and checker |
| ALCE, Gao et al., EMNLP 2023 ([arXiv 2305.14627](https://arxiv.org/abs/2305.14627)) | Citation recall and citation precision | An NLI model checks whether cited passages entail each statement | Defines the citation metrics. "Even the best models lack complete citation support 50% of the time" on ELI5 |
| TREC 2024 RAG nuggets, Pradeep et al., SIGIR 2025 ([arXiv 2504.15068](https://arxiv.org/html/2504.15068v1)) | Share of vital facts ("nuggets") present in an answer | LLM creates nuggets and assigns them. The primary metric `V_strict` counts fully supported vital nuggets. Run-level Kendall tau of fully automatic against fully manual scoring is 0.727 | Our key facts are hand-written nuggets. Run-level ranking agreement is what a bake-off needs |
| TREC 2024 RAG support, Thakur et al., SIGIR 2025 ([arXiv 2504.15205](https://arxiv.org/abs/2504.15205)) | Whether cited passages support answer sentences | GPT-4o against human assessors | 56% exact agreement on a 3-level scale from scratch, 72% after post-editing |
| UAEval4RAG, Peng et al., ACL 2025 ([arXiv 2412.12300](https://arxiv.org/abs/2412.12300)) | Handling of unanswerable questions | Six-category taxonomy, generated per knowledge base, scored by unanswered ratio and acceptable ratio | Use its taxonomy to write the must-refuse questions. It reports a trade-off between accuracy on answerable and refusal on unanswerable questions |
| Error bars for evals, Miller 2024 ([arXiv 2411.00640](https://arxiv.org/html/2411.00640v1)) | Statistics for eval reporting | Standard errors, clustered errors, resampling, paired differences, power analysis | Basis of the variance rules in section 7 |

### Benchmarks

| Benchmark | Content | License | Fit for this project |
| --- | --- | --- | --- |
| CRAG ([arXiv 2406.04744](https://arxiv.org/html/2406.04744v2)) | 4,409 QA pairs, 5 domains, mock web and knowledge-graph APIs. Scores perfect 1, acceptable 0.5, missing 0, incorrect -1. LLM auto-eval agreed with humans at 94.7% (ChatGPT) and 98.9% (Llama 3 70B) F1 | CC BY-NC 4.0 ([repo](https://github.com/facebookresearch/CRAG)) | Not usable: non-commercial, and its corpus is web pages. Its scoring scheme is reused in section 6 |
| RAGBench ([arXiv 2407.11005](https://arxiv.org/abs/2407.11005)) | About 100,000 examples from 5 industry domains, with labels for its TRACe metrics. A fine-tuned RoBERTa beat LLM judges | CC BY 4.0 ([dataset](https://huggingface.co/datasets/galileo-ai/ragbench)) | English judge calibration only |
| FRAMES ([arXiv 2409.12941](https://arxiv.org/abs/2409.12941)) | 824 English multi-hop questions over Wikipedia. Accuracy 0.40 without retrieval, 0.66 with multi-step retrieval | Apache-2.0 ([dataset](https://huggingface.co/datasets/google/frames-benchmark)) | Low. Wikipedia is not a GitHub source |
| MEMERAG, ACL 2025 ([arXiv 2502.17163](https://arxiv.org/html/2502.17163v4)) | 250 questions per language including German, native-speaker faithfulness labels | CC BY-NC 4.0 ([repo](https://github.com/amazon-science/MEMERAG)) | The only German judge meta-evaluation set found. Non-commercial, so legal must approve before use |
| BEIR ([arXiv 2104.08663](https://arxiv.org/abs/2104.08663)) | 18 English retrieval datasets. BM25 is "a robust baseline"; reranking models score best at high compute cost | Per dataset | Retrieval model choice, not a stack bake-off |
| MMTEB ([arXiv 2502.13595](https://arxiv.org/abs/2502.13595)) | Over 500 tasks in 250+ languages, including code retrieval. German retrieval tasks include GermanQuAD, GermanDPR, GerDaLIR, German government service and MIRACL ([mteb task list](https://github.com/embeddings-benchmark/mteb/tree/main/mteb/tasks/retrieval/deu)) | Per task. MIRACL is Apache-2.0, GermanQuAD is CC BY 4.0 | Embedding and reranker choice for German. It tests models, which the bake-off holds fixed |
| CodeRAG-Bench ([arXiv 2406.14497](https://arxiv.org/abs/2406.14497)) | Code generation with retrieval from library docs, tutorials, StackOverflow and GitHub. Retrievers struggle when the query shares few words with the code | See repo | Low. It measures code generation, not questions about a repo |
| SWE-QA, ACL 2026 Findings ([arXiv 2509.14635](https://arxiv.org/abs/2509.14635)) | 576 repository-level questions from real GitHub issues of 11 Python repos, including cross-file and multi-hop questions | Apache-2.0 ([repo](https://github.com/peng-weihan/SWE-QA-Bench)) | High. Real questions about real GitHub repos with reference answers |
| FreshStack ([arXiv 2504.13128](https://arxiv.org/abs/2504.13128)) | Retrieval over code and technical docs, community questions, nugget-level judgments. Off-the-shelf retrievers fall well short of oracles | CC BY-SA 4.0 ([queries](https://huggingface.co/datasets/freshstack/queries-oct-2024)) | Medium. English retrieval over GitHub-style corpora |

## 2. Metrics with and without a judge

| Metric | Needs a judge | How it is computed | Ready-made implementation |
| --- | --- | --- | --- |
| Retrieval recall at k | No | Share of gold citation IDs in the top k retrieved IDs | Ragas `IDBasedContextRecall`: "No LLM or embeddings" ([docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/)). Keiko `recallAtK` ([check-retrieval-quality.mjs](https://github.com/oscharko-dev/Keiko/blob/dev/scripts/check-retrieval-quality.mjs), line 460) |
| MRR | No | 1 divided by the rank of the first gold ID | Keiko `reciprocalRank` (same file, line 454). TruLens `GroundTruthAgreement.mrr` matches chunk text exactly (`if chunk in golden_chunks`, [groundtruth.py](https://github.com/truera/trulens/blob/main/src/feedback/trulens/feedback/groundtruth.py)), so it breaks when chunking differs between stacks. Ragas has no MRR |
| Retrieval precision at k, nDCG at k | No | From gold IDs | Ragas `IDBasedContextPrecision` ([source](https://github.com/explodinggradients/ragas/blob/main/src/ragas/metrics/_context_precision.py)); Keiko `binaryNdcgAtK` |
| Citation membership | No | Every citation in the answer points at a passage that the stack sent to the model | Keiko `reconcileInlineCitations` ([ADR-0144](https://github.com/oscharko-dev/Keiko/blob/dev/docs/adr/ADR-0144-grounded-entailment-stage.md), context section) |
| Citation precision and recall against gold | No | Cited IDs (path plus overlapping line range, or issue or PR number) compared with gold IDs | Own code, about 20 lines |
| Refusal | No | A must-refuse question gets a refusal and states no value; an answerable question is not refused | Keiko MemoriaViva scorer: a negation marker (German `nicht`, `kein*`; English `not`, `cannot`) and no value-shaped string ([memoria-viva-lab.md](https://github.com/oscharko-dev/Keiko/blob/dev/docs/qa/memoria-viva-lab.md), "Scoring") |
| Key-fact recall | No | Share of the gold key facts (term groups) found in the answer, with author-set German stems (`Tabelle*` matches Tabelle, Tabellen, Tabellenform) | Keiko `score.mjs` rules, same document |
| Citation support | Yes | Each claim is entailed by the passage it cites | Keiko `reconcileClaimEntailment` with an `EntailmentJudge` returning `supported`, `unsupported` or `unavailable` (ADR-0144 D1, D2). DeepEval community `CitationFaithfulnessMetric` ([docs](https://github.com/confident-ai/deepeval/blob/main/docs/content/docs/(community)/metrics-citation-faithfulness.mdx)) |
| Faithfulness | Yes | Share of answer claims entailed by the retrieved context | Ragas `Faithfulness`, two LLM calls ([source](https://github.com/explodinggradients/ragas/blob/main/src/ragas/metrics/collections/faithfulness/metric.py)). DeepEval `FaithfulnessMetric`. TruLens groundedness |
| Semantic answer correctness | Yes | Claim-level precision and recall of the answer against the reference answer | Ragas `FactualCorrectness` with `mode="f1"` ([source](https://github.com/explodinggradients/ragas/blob/main/src/ragas/metrics/collections/factual_correctness/metric.py)). Ragas `AnswerCorrectness` adds embedding similarity unless `weights=[1.0, 0.0]` |

Ragas's LLM-based `ContextRecall` checks claims of the reference answer against retrieved text with an LLM. With gold citation IDs, the ID-based variant answers the same question without a judge. Keiko's deterministic gates enforce floors of the same shape: recall at 5 at least 90%, MRR at least 0.90, and 1.0 for fabricated-citation detection ([grounded-certification-baseline.md](https://github.com/oscharko-dev/Keiko/blob/dev/docs/qa/grounded-certification-baseline.md)).

The earlier note [`rag-stack-landscape.md`](https://github.com/Niko4417/OPT-cc/blob/research/rag-stack-landscape/docs/research/rag-stack-landscape.md) says citation metrics are not built into Ragas or DeepEval. DeepEval now ships `CitationFaithfulnessMetric` as a community metric, and Ragas ships `QuotedSpansAlignment`, a verbatim check of quoted spans ([source](https://github.com/explodinggradients/ragas/blob/main/src/ragas/metrics/collections/quoted_spans/metric.py)).

## 3. Pointing each framework at LiteLLM

The proxy listens on loopback, for example `http://127.0.0.1:4000/v1`. Each route alias (`answer`, `judge`, `embed`, `rerank`) declares its model location as on-device.

**Ragas** (Apache-2.0, v0.4.4, 2026-11-14). `llm_factory(model, provider="openai", client=..., mode=..., **kwargs)` takes a pre-built OpenAI client ([llms/base.py](https://github.com/explodinggradients/ragas/blob/main/src/ragas/llms/base.py), line 624). An `AsyncOpenAI(base_url="http://127.0.0.1:4000/v1", api_key=<LiteLLM key>)` client routes every judge call through LiteLLM. The docstring says to use `Mode.MD_JSON` "for backends that don't support response_format parameter". `embedding_factory("litellm", model=..., api_base=...)` does the same for embeddings ([customize_models.md](https://github.com/explodinggradients/ragas/blob/main/docs/howtos/customizations/customize_models.md)). Ragas defaults to `timeout=180` and `max_workers=16` ([run_config.py](https://github.com/explodinggradients/ragas/blob/main/src/ragas/run_config.py)). Keiko serves one decode at a time ([offline gate record](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/docs/qa/review-agent-offline-gate-2026-11-01.md), "Serving"), so 16 queued judge calls would exceed 180 seconds. Set `max_workers` to 1 or 2 and `timeout` to 900. Ragas posts usage data to `https://t.explodinggradients.com` unless `RAGAS_DO_NOT_TRACK=true` ([_analytics.py](https://github.com/explodinggradients/ragas/blob/main/src/ragas/_analytics.py), lines 37 to 49 and 233).

**DeepEval** (Apache-2.0, v4.1.x). `deepeval set-local-model --model=<alias> --base-url="http://127.0.0.1:4000/v1/"`, or `LocalModel(model=..., base_url=..., api_key=..., temperature=0, format="json")` in code ([vllm.mdx](https://github.com/confident-ai/deepeval/blob/main/docs/content/integrations/models/vllm.mdx)). `LiteLLMModel(model="openai/<alias>", base_url=...)` also works ([LiteLLM integration](https://deepeval.com/integrations/models/litellm)). The `hybrid` and `system_one` eval modes call Jev, a hosted TypeSafe AI model, and "need a TypeSafe AI key" ([evaluation-eval-modes.mdx](https://github.com/confident-ai/deepeval/blob/main/docs/content/docs/evaluation-eval-modes.mdx)). Keep the default `llm` mode. DeepEval sends PostHog telemetry unless `DEEPEVAL_TELEMETRY_OPT_OUT=1` ([data-privacy.mdx](https://github.com/confident-ai/deepeval/blob/main/docs/content/docs/data-privacy.mdx)).

**TruLens** (MIT, 2.16.0, 2026-10-19). `trulens.providers.litellm.LiteLLM(model_engine=..., api_base=...)` ([provider.py](https://github.com/truera/trulens/blob/main/src/providers/litellm/trulens/providers/litellm/provider.py), lines 36 to 101). For a LiteLLM proxy alias the engine is `openai/<alias>`. TruLens is built around tracing an app and a dashboard. That adds little to a batch run over a fixed gold set, and tracing would need to be wired into each candidate stack.

**RAGChecker** (Apache-2.0) reaches the proxy through RefChecker's `api_base`. It is a good second opinion on retriever and generator errors, but the code has not changed since 2024-12.

Ragas fits best. It has claim-level `Faithfulness` and `FactualCorrectness`, a structured-output client for any OpenAI-compatible URL, ID-based retrieval metrics and a German prompt adaptation (section 5). DeepEval works too, with more cloud hooks to switch off.

## 4. Judge reliability at the lab pair's size

No benchmark I found reports judge agreement for Qwen3.6-35B-A3B or Gemma 4 31B. The LLM-AggreFact leaderboard of grounding checkers lists no Qwen3 or Gemma row ([leaderboard](https://llm-aggrefact.github.io/), read 2026-11-17). The closest evidence:

| Source | Judge | Task | Result |
| --- | --- | --- | --- |
| MEMERAG ([arXiv 2502.17163](https://arxiv.org/html/2502.17163v4), Table 10) | Qwen 2.5 32B | German RAG faithfulness against native-speaker labels, English judge prompt | 76.79% balanced accuracy (GPT-4o mini 73.58%, Llama 3.2 11B 59.91%) |
| Same, Table 2 | Three human annotators | German faithfulness | Fleiss kappa 0.81, Gwet's AC1 0.84 |
| LLM-AggreFact ([leaderboard](https://llm-aggrefact.github.io/)) | QwQ-32B-Preview, Qwen2.5-72B, GPT-4o | Grounded fact checking over 11 datasets, balanced accuracy | 71.8, 75.6, 75.9 |
| Same | FactCG-DeBERTa-L (0.4B), MiniCheck-Flan-T5-L (0.8B), Granite Guardian 3.3 (8B) | Same | 75.6, 75.0, 76.5 |
| Judging the Judges ([arXiv 2406.12624](https://arxiv.org/abs/2406.12624)) | 13 judges | Answer grading | Only the largest judges reach "reasonable alignment"; smaller judges still rank systems usefully; judges lean lenient |
| Multilingual LLM-as-a-judge ([arXiv 2505.12201](https://arxiv.org/abs/2505.12201)) | 5 model families, 25 languages | The same item judged in several languages | Average Fleiss kappa about 0.3 across languages |
| Self-preference ([arXiv 2404.13076](https://arxiv.org/abs/2404.13076)) | GPT-4, Llama 2 | Judging own output | Self-recognition correlates with self-preference |

What follows for the lab pair:

- A 31B dense judge should land near the 32B-class results: about 72% to 77% balanced accuracy on claim support. Dedicated 0.4B to 8B checkers reach the same range on English data, so judge size is not the main limit.
- Judges rank systems better than they score single answers (Judging the Judges; TREC nugget run-level tau 0.727). A bake-off needs rankings, so a local judge is adequate for it.
- Gemma judging Qwen's answers avoids self-preference. Keiko uses the same split: Qwen writes reviews and Gemma counter-checks them ([lab README](https://github.com/oscharko-dev/Keiko/blob/9c3329099be42e82a8fb5d95e0468be4550aed71/scripts/testing/review-quality/lab/README.md)).
- Small checkers are not an option here. HHEM-2.1-Open (Ragas `FaithfulnesswithHHEM`) is tagged English only, and the German HHEM-2.3 is "available exclusively via Vectara's platform" ([model card](https://huggingface.co/vectara/hallucination_evaluation_model)). Bespoke-MiniCheck-7B needs a commercial license from Bespoke Labs ([MiniCheck README](https://github.com/Liyan06/MiniCheck)).

## 5. German

- Ragas `prompt.adapt(target_language="german", llm=llm)` translates the few-shot examples with the judge. `adapt_instruction=True` also translates the instructions. `Faithfulness` and `FactualCorrectness` each have two prompts to adapt ([metrics_language_adaptation.md](https://github.com/explodinggradients/ragas/blob/main/docs/howtos/customizations/metrics/metrics_language_adaptation.md)). Adapt once, review by hand, and keep the reviewed prompts fixed so that runs stay comparable.
- MEMERAG judged German answers with English prompts and got its best German score that way. English instructions with German data are a valid default.
- Gemma 4 31B states "out-of-the-box support for 35+ languages, pre-trained on 140+ languages" and 88.4% on MMMLU ([model card](https://huggingface.co/google/gemma-4-31B-it)). The Qwen3.6-35B-A3B card gives no language count or multilingual QA score ([model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)).
- Key-fact matching needs German stems written by the gold-set author. Keiko's scorer missed "Tabellenform" with a whole-word `Tabellen`, and Keiko answered in German even to English probes ([memoria-viva-lab.md](https://github.com/oscharko-dev/Keiko/blob/dev/docs/qa/memoria-viva-lab.md)). The gold set states the expected answer language per question.

## 6. Metric set for this project

The gold set holds, per question: the question, the expected answer language, gold citation IDs (repo path plus line range, or issue or PR number), a short reference answer, 1 to 5 vital key facts as term groups (nuggets in the TREC sense), a question type, and a must-refuse flag.

| Area | Metric | Judge | Role in the bake-off |
| --- | --- | --- | --- |
| Retrieval | Recall at k against gold citation IDs, with k equal to the shared context budget (section 7) | No | Primary |
| Retrieval | MRR against gold citation IDs | No | Primary |
| Citation correctness | Fabricated-citation rate: citations outside the evidence sent to the model | No | Gate: must be 0 |
| Citation correctness | Citation recall and precision against gold IDs | No | Primary |
| Citation correctness | Citation support: share of cited claims entailed by their cited passage (ALCE citation precision) | Gemma, per claim | Primary |
| Faithfulness | Ragas `Faithfulness` against all retrieved context | Gemma | Secondary, full profile only |
| Answer correctness | Key-fact recall: share of vital key facts present (`V_strict` style) | No | Primary |
| Answer correctness | Ragas `FactualCorrectness`, `mode="f1"`, against the reference answer | Gemma | Secondary, full profile only |
| Refusal | Correct refusals on must-refuse questions; false refusals on answerable ones | No | Primary, reported as two rates |
| Summary | CRAG-style score per question: +1 if all vital facts are present and no fabricated citation, 0 for a refusal, -1 otherwise; the mean over all questions | No | Headline number. It rewards refusing over guessing |
| Latency | End-to-end seconds per question, p50 and p95; time to first token, p50 | No | Primary. Indexing time of the corpus is reported once per stack |

Each stack must expose its ranked retrieved IDs and its citations per answer. If a stack shows only cited sources, its retrieval metrics are computed from the trace of its retriever calls or reported as missing. They are never estimated.

## 7. Bake-off protocol

**Held fixed across stacks.**

- One Mac, one macOS build, nothing else memory-heavy running. Record chip, RAM, mlx-lm and LiteLLM versions and model revisions, as Keiko's lab pins them.
- The same LiteLLM routes for every stack: `answer` is `Qwen3.6-35B-A3B-4bit` in non-thinking mode with the card's instruct sampling (temperature 0.7, top-p 0.8, top-k 20); `judge` is `gemma-4-31b-it-4bit` without thinking at temperature 0; `embed` and `rerank` are the chosen local embedding model and reranker. LiteLLM exposes a Cohere-compatible `/rerank` endpoint ([docs](https://docs.litellm.ai/docs/rerank)). A stack that cannot use a route is run with its closest option, and the deviation is recorded.
- The same corpus snapshot: one commit SHA per repo, and the same issue and pull request export.
- The same chunking budget where configurable: one chunk size and overlap, and one context budget, for example 8 passages or 4,000 tokens. A stack with fixed chunking keeps its own, and the report says so.
- Each stack's own answer prompt, because the prompt is part of the stack. Each stack gets the same added instruction to cite sources and to refuse when the sources do not answer.
- The same gold set and the same scoring script.

**Gold set.** 50 questions to start: German and English mixed; spread over code, docs, decision records (including superseded ones), workshop results, issues and pull requests; single-fact, multi-hop, "which decision is current" and "where in the code" questions; and 5 to 8 must-refuse questions written with the UAEval4RAG categories. Test data is public or synthetic.

**Runs.**

1. Index the corpus once per stack and record the indexing time.
2. One warm-up question per stack, not scored.
3. Three repetitions of all 50 questions per stack, because the answer model samples at temperature 0.7. Repetitions resample answers, as Miller recommends. Interleave stacks question by question (A, B, C, A, B, C) to spread thermal and memory drift evenly.
4. Deterministic metrics and per-claim citation support on all repetitions. Ragas `Faithfulness` and `FactualCorrectness` on the first repetition only.
5. Judge self-consistency: judge 20 answers twice and report the share of identical verdicts.
6. Judge calibration, once: label claim support for 50 answers by hand and report the judge's balanced accuracy and Cohen's kappa. If both disagree with the human labels on which stack leads, the judge-based metrics do not decide.

**Run time** for 3 stacks: 450 answers at about 8 seconds is 1 hour; per-claim citation support at about 20 seconds per answer is 2.5 hours; the two Ragas metrics on 150 answers at about 160 seconds is 6.7 hours. The total is about 10 hours, one night, plus indexing.

**Reporting variance** ([Miller 2024](https://arxiv.org/html/2411.00640v1)).

- Average each question over its three repetitions first. Report each metric as the mean with its standard error over questions.
- Compare stacks on per-question paired differences with a 95% confidence interval (paired bootstrap with 10,000 resamples, or a paired t-test). Do not compare two means with overlapping error bars by eye.
- Use clustered standard errors when several questions draw on the same source document.
- Report the spread between repetitions per stack, to show how much comes from sampling.
- Power: the minimum detectable paired difference at 80% power and alpha 0.05 is about 2.8 times the standard deviation of per-question differences divided by the square root of n. With a standard deviation of 0.3 and 50 questions, that is about 12 points. Detecting 5 points needs about 280 questions. Measure the real standard deviation in the first run, and grow the gold set if the stacks land closer than the detectable difference.

## 8. Public benchmark subsets worth adding

| Subset | Why | Use |
| --- | --- | --- |
| SWE-QA, questions for 1 or 2 of its 11 repos (Apache-2.0) | Real developer questions about real GitHub repos, with reference answers and cross-file questions. It tests code questions, the weakest part of most document RAG stacks | Add next to the own gold set. Each stack indexes those repos. Score with `FactualCorrectness` and refusal; there are no gold citation IDs |
| GermanQuAD or MIRACL German retrieval from MMTEB (CC BY 4.0, Apache-2.0) | German retrieval with public labels | Choose the embedding model and reranker before the bake-off. Not part of the bake-off, because the bake-off holds those models fixed |
| RAGBench (CC BY 4.0) | Human-quality support labels | English judge calibration if hand labels are scarce |
| MEMERAG German (CC BY-NC 4.0) | German faithfulness labels, the only German judge meta-evaluation found | German judge calibration, only after a license check, because the license is non-commercial |

Leave out CRAG (non-commercial, web corpus), FRAMES (Wikipedia, English), BEIR (English retrieval models only), CodeRAG-Bench (code generation) and FreshStack (English retrieval only; an option if code retrieval decides between stacks).

## 9. Run time for 50 questions on a 64 GB Mac

Measured speeds:

- Qwen3.6-35B-A3B 4-bit on an M5 Pro (48 GB): 2,165 tokens per second prefill and 91.3 tokens per second generation at 4k context, 25.1 GB peak ([oMLX benchmark](https://omlx.ai/benchmarks/mrpdgltl)).
- gemma-4-31b-it 4-bit on an M5 Pro (48 GB, 16 GPU cores): 307 tokens per second prefill and 13.0 tokens per second generation at 4k context ([oMLX benchmark](https://omlx.ai/benchmarks/a6tzwsde)). Keiko measured about 12 to 13 generated tokens per second for the same model on an M5 Pro with 64 GB under mlx-lm 0.32.2, about 80% of them reasoning ([offline gate record](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/docs/qa/review-agent-offline-gate-2026-11-01.md)).
- Both 4-bit models are 38.8 GB of weights. Keiko's lab waits for 42 GB of free memory before it starts the pair ([lab README](https://github.com/oscharko-dev/Keiko/blob/9c3329099be42e82a8fb5d95e0468be4550aed71/scripts/testing/review-quality/lab/README.md)), so the pair plus a small embedding model and reranker fit on 64 GB.

Assumptions per question: 5,000 prompt tokens for the answer, a 300-token answer, a 100-token reference answer, about 5 claims per answer, and no thinking for either model.

| Step | Per question | 50 questions |
| --- | --- | --- |
| Retrieval, reranking, answer by Qwen (2.4 s prefill, 3.3 s generation, about 2 s retrieval) | about 8 s | about 7 min |
| Deterministic metrics | under 0.1 s | seconds |
| Per-claim citation support: 5 Gemma calls of about 500 tokens in and 30 out | about 20 s | about 17 min |
| Ragas `Faithfulness` by Gemma: 1,000 in and 250 out, then 5,500 in and 400 out | about 70 s | about 1 h |
| Ragas `FactualCorrectness` in `f1` mode by Gemma: 2 decompositions and 2 checks | about 90 s | about 1.2 h |

These are estimates from the speeds above, not a measured run. The first run records real wall time per step. With Gemma's thinking on, generated tokens grow about fivefold, and the Ragas pass alone would take 9 to 10 hours. Qwen as the judge would cut the Ragas pass to about 20 minutes, but it would grade its own answers.

## 10. Recommended setup

1. A gold set of 50 questions as in section 6, stored in the repo as JSON lines.
2. A small harness outside the app. It calls each stack's question API, stores answer, citations, retrieved IDs and timings, and computes the deterministic metrics. Port Keiko's `reciprocalRank`, `recallAtK`, citation membership and MemoriaViva term matching (Apache-2.0).
3. Judge metrics: per-claim citation support in Keiko's `EntailmentJudge` shape for every run, plus Ragas 0.4.x `Faithfulness` and `FactualCorrectness(mode="f1")` for the full profile. The judge is `gemma-4-31b-it-4bit` through the LiteLLM `judge` alias, thinking off, temperature 0, `max_workers` 1 or 2, `timeout` 900.
4. No cloud calls: `RAGAS_DO_NOT_TRACK=true`, `DEEPEVAL_TELEMETRY_OPT_OUT=1` if DeepEval is used, every LiteLLM route on-device, and an on-device-only model policy.
5. Two profiles. The quick profile (deterministic metrics plus citation support, about 25 minutes) runs after each change. The full profile adds the Ragas metrics (about 2.5 hours). The bake-off of section 7 runs overnight. The process script starts and stops the model server, LiteLLM and the stack under test, like Keiko's `lab.sh run`.
6. Calibrate the judge once against hand labels before any judge-based metric decides between stacks.
