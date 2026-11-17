# Quality gates, CI and evaluation checks for the project memory

Research for [issue #14](https://github.com/Niko4417/OPT-cc/issues/14), map [issue #1](https://github.com/Niko4417/OPT-cc/issues/1). Sources were read on 2026-11-17. It builds on [`docs/research/local-evaluation.md`](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md) (branch `research/local-evaluation`, issue #7), which chose the metric set, the judge and the bake-off protocol. This note does not repeat them.

Keiko links point at commits of [oscharko-dev/Keiko](https://github.com/oscharko-dev/Keiko) (Apache-2.0): `dev` at `d72d6db` and the review-quality branch `feat/epic-4130-keiko-for-quality` at `fc5d65a`. The fork [Niko4417/Keiko](https://github.com/Niko4417/Keiko) has the same code history, but its `dev` branch stops at 2026-06-09 and it has none of the 4130 branches, so the files below exist only upstream.

## Question

Which quality gates, CI checks and evaluation runs should the project memory have? Which of them can be taken over from Keiko? What do RAG projects and papers recommend for regression-testing answer quality in CI: retrieval and citation regression sets, judge checks with a local model, thresholds, flakiness handling, and run time per pull request against nightly runs?

## Short answer

- Split the checks by what they need. Checks without a language model run on every pull request on a GitHub-hosted runner in under 10 minutes. Checks that need the lab pair run nightly on the V1 Mac, because the largest GitHub-hosted Mac has 14 GB of RAM and the pair needs 38.8 GB.
- Do not use a self-hosted runner on the Mac for this public repo. GitHub says self-hosted runners "should almost never be used for public repositories". The Mac runs the nightly job itself, from the start script, on `main` only.
- Keiko has 8 grounding and retrieval gates in its required CI. None of them calls a model: each drives the real pipeline over synthetic fixtures with scripted embedding, reranking and judge ports. Together they take about 1.7 minutes. No Keiko workflow runs a real model; the model-backed review-quality lab and MemoriaViva lab run by hand on a Mac.
- Keiko's gate code imports Keiko's own packages, so it transfers only if Keiko wins the stack choice. Its methods transfer to any stack: floors of 1.0 on fixtures, an injected regression that must turn the gate red, a judge that is "unavailable" counts as a warning and never as a pass, a scripted OpenAI-compatible model for offline runs, and "unmeasured" as a third verdict next to pass and fail.
- Put the gates in the evaluation harness outside the app, as issue #7 proposed. The harness calls the stack's question API, so the same gates work for Keiko, Open WebUI and LightRAG.
- Handle judge flakiness with five rules: judge at temperature 0 with one request at a time; cache verdicts by input; run seeded canaries through the real judge every night; treat a run with more than 5% failed calls as unmeasured; and gate only on paired statistics against a committed baseline, confirmed by a second run before anything turns red. A judge metric never blocks a pull request.
- Ragas has no CI gate of its own. DeepEval (`deepeval test run`) and promptfoo have thresholds and repeats, but both judge one run in isolation. A 50-question gold set needs a paired comparison with a baseline, so the harness should own that rule.

## 1. Keiko's quality gates

### 1.1 Grounding and retrieval gates in the required CI

All 8 run as steps of the `Core quality` job of [`ci.yml`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/.github/workflows/ci.yml) (lines 458 to 527) on `ubuntu-latest`, on every pull request to `dev`. Run times are the step times of [CI run 44110775652](https://github.com/oscharko-dev/Keiko/actions/runs/44110775652) on 2026-11-17. Floors and fixture counts come from the scripts and from [grounded-certification-baseline.md](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/qa/grounded-certification-baseline.md).

| Gate (npm script) | What it checks | Fixtures and floors | Needs | Run time | Transfers to another stack |
| --- | --- | --- | --- | --- | --- |
| `check:retrieval-quality` ([script](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-retrieval-quality.mjs)) | Lexical repo search: top-1 file, recall at 5, MRR, nDCG at 5, line-level evidence, leaks of generated files and prose decoys | 15 workspace cases with every floor at 1 and 0 leaks ([budget](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-retrieval-quality.budget.json)); 29 Local Knowledge fixtures with recall 0.9, precision 0.8, MRR 0.9, nDCG 0.9 | In-memory synthetic repos, compiled Keiko packages | 11 s | Method only. The metric helpers (`binaryNdcgAtK`, `recallAtK`, `reciprocalRank`) are small pure functions worth porting |
| `check:grounded-retrieval-quality` ([script](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-grounded-retrieval-quality.mjs)) | The production path: semantic search, RRF fusion and model reranking | 10 distractor-dense cases; top-1 at least 0.8, recall at 3 at least 0.9, nDCG at 3 at least 0.85, citation support at least 0.8 ([floors](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-server/src/grounded-retrieval-eval.ts), line 757). Two injected regressions, `reranker-reversed` and `embedding-flat`, must fall below the floors | A scripted embedding port with topic vectors and a scripted rerank port; no model | 4 s | Method only; it calls Keiko server internals |
| `check:grounded-faithfulness` ([script](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-grounded-faithfulness.mjs)) | Citation reconciliation and abstention over scripted answers: faithful, fabricated citation, confident answer over empty evidence, refusal | 12 fixtures; every rate at 1.0 | Scripted answerer | 2 s | Method transfers if the glue code reconciles citations; the code is Keiko's |
| `check:grounded-entailment` ([script](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-grounded-entailment.mjs)) | Claim segmentation and per-claim citation support: unsupported claims flagged, supported claims clean, an unavailable judge gives a warning, a pass-through judge must fail | 15 fixtures; every rate at 1.0 and the non-tautology proof required | Scripted judge behind the same `EntailmentJudge` port as the real judge ([ADR-0144](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/adr/ADR-0144-grounded-entailment-stage.md)) | 2 s | Method transfers; the plumbing test is cheap to rebuild around our judge call |
| `check:grounded-retrieval-latency` ([script](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-grounded-retrieval-latency.mjs)) | Code-path time of retrieval, reranking and entailment with stubbed models | p50 at most 400 ms and p95 at most 800 ms, against 35 and 37 ms observed; an injected judge delay of 1,300 ms must breach p95 ([budget](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/check-grounded-retrieval-latency.budget.json)) | Stubs; 3 warm-up and 12 timed runs | 11 s | Method transfers |
| `check:retrieval-latency` | Lexical search time over 2,000 synthetic files | p95 at most 500 ms against 16 ms observed | Stubs | 6 s | Method only |
| `check:knowledge-m2-closeout` | Re-runs 6 retrieval proofs, including the USearch ANN index, and compares a freshly rendered evidence file with the committed one | Fails on any drift | USearch native runtime, fetched with a pinned SHA-256 | 62 s | No; Keiko-specific |
| `check:context-quality` | Context allocation invariants: fact preservation, budget overflow, prompt-injection isolation, redaction, line references | Hard invariants plus a soft latency ceiling | Offline scenario corpus | 5 s | No; Keiko's agent context code |

The same `Core quality` job also runs type checks, lint (12 minutes 52 seconds in that run) and about 17 repo hygiene checks, and the job takes 26 to 31 minutes ([`ci.yml`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/.github/workflows/ci.yml), comment at line 362). The grounding gates are about 1.7 minutes of it.

Two shared helpers carry the method. `evaluateFloors` fails on a missing or non-finite metric, not only on a low one. `runRegressionProbes` fails when an injected regression stays above the floors, when no probe ran, or when a selected probe was skipped ([quality-helpers.ts](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/packages/keiko-evaluations/src/quality-helpers.ts)). Both are under 100 lines and Apache-2.0.

### 1.2 Model-backed labs, outside CI

None of Keiko's 20 workflows starts MLX, Ollama or LiteLLM. A search of all workflow files on `dev` for `mlx`, `ollama`, `litellm`, `memory-lab` and `review-quality` finds only macOS runner names. The model-backed measurements run on a Mac by hand.

| Lab | What it measures | Needs | Run time | What transfers |
| --- | --- | --- | --- | --- |
| Review-quality lab, [`lab.sh`](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/scripts/testing/review-quality/lab/README.md) and [`run.mjs`](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/scripts/testing/review-quality/run.mjs) on the 4130 branch | Pull request reviews by Qwen with Gemma counter-checks, replayed on 30 historical pull requests, or on 13 seeded defects, 1 advisory seed and 3 clean controls | The 4-bit lab pair on `mlx_lm` 0.32.2 behind LiteLLM 1.106.6 at pinned revisions; 42 GB of free memory before it starts | 3.6 hours for 30 pull requests, 1 hour for the seeds, at 4 bit on an M5 Pro with 64 GB | Process control: wait for free memory, start the pair and the gateway, run, stop everything also on failure or Ctrl-C, restart a server whose generation thread died. Body-free evidence (`evidence.json`, `summary.md`) |
| Offline review gate, [ADR-0184 D9](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/docs/adr/ADR-0184-keiko-for-quality-governed-pull-request-review-agent.md) | At least 50% actionable findings on a chronological holdout, every mandatory seed found, at most 3 findings per pull request | Same lab | One holdout run on 2026-11-06, now spent | A holdout used once and never for tuning; "the threshold is not lowered"; a Wilson 95% interval next to every rate; a run with more than 20% incomplete reviews fails on coverage |
| MemoriaViva lab, [memoria-viva-lab.md](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/qa/memoria-viva-lab.md) | What the memory feature learns and answers, through the public routes | Offline: a scripted OpenAI-compatible model ([`scripted-model.mjs`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/testing/memory-lab/scripted-model.mjs)). Live: a local endpoint, 3 repetitions | Offline curated set runs inside `npm test`; the document gives no live run time | Three arms per run: `off` (floor), `product`, `oracle` (truth handed in, so product minus oracle separates extraction loss from retrieval and answer loss). A failed unit is left out of every average and the run exits 5 ("unmeasured"). [`compare.mjs`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/scripts/testing/memory-lab/compare.mjs) calls no difference progress without at least 3 repetitions per run |

The review lab also shows where local-model runs break. In one run Gemma's server ran out of GPU memory, its generation thread died, it "answered every later request with 404", and the remaining 24 reviews ended `model-failed` ([offline gate record](https://github.com/oscharko-dev/Keiko/blob/fc5d65a395cd62c237491c3a69feeedf244e2315/docs/qa/review-agent-offline-gate-2026-11-01.md), around line 145). After that the lab serves one request at a time, prefills in 512-token steps and caps the prompt caches at 4 GB (Qwen) and 6 GB (Gemma). Issue #4 also caps the prompt caches, at 2 and 4 GB.

### 1.3 CI rules worth copying

- A measurement lane measures and a separate gate judges. Keiko's nightly performance lane asserted its own budgets, so an overrun produced no evidence at all, and it failed 12 of 12 runs unnoticed ([ADR-0156](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/adr/ADR-0156-measurement-and-verdict-separation.md)). The nightly evaluation should always write its scorecard, and a separate check should compare it with the baseline.
- Absolute time budgets belong to one reference machine. Keiko's budgets were measured on a 14-core arm64 machine and a 4-core hosted runner "can never" meet them, so the hosted nightly lane only checks digest drift ([`nightly-perf-evidence.yml`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/.github/workflows/nightly-perf-evidence.yml)). Latency and indexing time for the project memory are therefore measured on the V1 Mac only.
- Re-run infrastructure failures, never test verdicts. Keiko re-runs a failed run at most once, and only when every failed job ran zero steps under a known runner-failure signature ([ADR-0161](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/adr/ADR-0161-bounded-re-run-for-infrastructure-signature-failures.md)).
- Nightly suites are not required checks. Keiko's extended end-to-end suite runs nightly and "a nightly red is a signal to fix, not a merge blocker" ([`e2e-extended.yml`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/.github/workflows/e2e-extended.yml)).
- Secret scanning with a checksum-pinned Gitleaks binary over the pull request range ([`ci.yml`](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/.github/workflows/ci.yml), job `secret-scan`) transfers as is. Most other Keiko CI jobs (verified-tree reuse, SonarCloud, coverage shards, merge-candidate checks) serve a large product repo and are more than a glue repo needs.

## 2. How RAG projects and papers regression-test answer quality

### 2.1 Tools

| Tool | CI mechanism | Repeats and caching | Fit |
| --- | --- | --- | --- |
| Ragas 0.4 (Apache-2.0) | None. Its RAG guide prints a pass rate ("43/66 passed") and leaves the cutoff to the user ([guide](https://docs.ragas.io/en/stable/howtos/applications/evaluate-and-improve-rag/)) | `DiskCacheBackend` passed to `llm_factory(cache=...)` caches every judge call by prompt ([caching how-to](https://docs.ragas.io/en/stable/howtos/customizations/_caching/)) | Use for the judge metrics, with the gate rule in the harness |
| DeepEval 4.2 (Apache-2.0) | `deepeval test run` runs pytest-style tests; "Failing metrics fail the build" ([CI guide](https://deepeval.com/docs/evaluation-unit-testing-in-ci-cd)) | `-r` repeats each case, `-c` reads cached results keyed by test case and metric configuration, `-i` ignores metric errors ([flags](https://deepeval.com/docs/evaluation-flags-and-configs)). A metric marked `flaky=True` turns a failure into a warning | Usable, but `flaky=True` and `-i` hide regressions instead of measuring them |
| promptfoo 0.125 (MIT) | A test case fails when "the combined score of assertions is less than" its `threshold`; a GitHub Action compares before and after when a pull request changes a prompt ([reference](https://www.promptfoo.dev/docs/configuration/reference/), [action](https://www.promptfoo.dev/docs/integrations/github-action/)) | `evaluateOptions.repeat` (default 1); disk cache on by default | Built for prompt changes; less fit for a fixed RAG app behind its own API |

All three score one run against fixed per-case thresholds. None compares a run with a stored baseline on the same questions, which is the comparison a small gold set needs (section 2.2).

### 2.2 Papers on noise and statistics

| Source | Finding | Consequence here |
| --- | --- | --- |
| Atil et al., [arXiv 2408.04667](https://arxiv.org/abs/2408.04667) | Five LLMs in "deterministic" settings, 8 tasks, 10 runs: accuracy varied "up to 15%" across runs, and no model gave repeatable accuracy on all tasks | One nightly run is not a measurement. Repeat |
| Thinking Machines, [Defeating nondeterminism in LLM inference](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) | 1,000 completions at temperature 0 from Qwen3-235B gave 80 distinct outputs; the cause is batch size changing with server load. Batch-invariant kernels made all 1,000 identical | Serve the judge one request at a time, as Keiko's lab does, and measure how often it repeats itself |
| Yuan et al., [arXiv 2506.09501](https://arxiv.org/abs/2506.09501) | Batch size, GPU count and GPU type change greedy outputs through floating-point error | Results from different machines are not comparable; baselines are per machine |
| Haldar and Hockenmaier, Rating Roulette, [arXiv 2510.27106](https://arxiv.org/abs/2510.27106) | LLM judges show "low intra-rater reliability" across runs, "almost arbitrary in the worst case"; removing sampling stabilises scores but lowers agreement with humans | Use binary verdicts per claim, not 1 to 10 scores, and measure self-agreement |
| [arXiv 2408.09235](https://arxiv.org/abs/2408.09235) (Reference-Guided Verdict) | Judges were consistent over 5 repeated runs at temperature 0 with binary prompts; open-ended prompts varied more | Same: binary claim-support verdicts are the stable form |
| Bowyer et al., ICML 2025, [arXiv 2503.01747](https://arxiv.org/html/2503.01747) | Below a few hundred items, CLT error bars are too narrow. For one rate use Wilson or Bayesian intervals; for two systems on the same questions use a paired method | 50 questions is in that range. Use Wilson intervals for rates and a paired comparison for differences |
| Miller, [arXiv 2411.00640](https://arxiv.org/abs/2411.00640) | Paired differences, clustered errors and resampled answers | Already the protocol of issue #7 |
| Madaan et al., [arXiv 2406.10229](https://arxiv.org/abs/2406.10229) | Score differences between models "may not be meaningful" without the seed variance | Keep a noise band per metric from the baseline repetitions |

Issue #7 computed that 50 questions detect paired differences of about 12 points at 80% power. A nightly gate on 50 questions therefore catches large regressions only. Smaller drops show up as a trend over several nights or need a larger gold set.

## 3. Where each check can run

| Place | Resources | Can run |
| --- | --- | --- |
| GitHub-hosted `ubuntu-latest`, public repo | 4 CPUs, 16 GB RAM ([runner reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)) | Everything without a language model. The 0.6B embedding model and reranker of issue #4 fit on CPU, but their speed there is not measured |
| GitHub-hosted macOS | 7 GB (standard arm64), 14 GB (M2 xlarge, the largest arm64 size), 30 GB (Intel large, no Apple GPU for MLX) ([larger runners](https://docs.github.com/en/actions/reference/runners/larger-runners)) | Not the lab pair, which needs 38.8 GB of weights |
| Self-hosted runner on the V1 Mac | 64 GB | Ruled out for a public repo: anyone can open a pull request that runs on it ([secure use](https://docs.github.com/en/actions/reference/security/secure-use)). `Niko4417/OPT-cc` is public |
| The V1 Mac, scheduled locally | 64 GB, the lab pair | Nightly and weekly model-backed runs on `main`. A `launchd` job calls the start script's run command, like Keiko's `lab.sh run`, and publishes a body-free scorecard |

## 4. Recommended gates

The gates live in the evaluation harness of issue #7, next to the app, and talk to the stack only through its question API and LiteLLM. They work the same for Keiko, Open WebUI and LightRAG. Where the chosen stack already has an equivalent (Keiko's grounding gates), keep it and add the harness gates on top.

### 4.1 Per pull request (GitHub-hosted Ubuntu, no language model, target under 10 minutes)

| Gate | What it checks | Threshold | Needs |
| --- | --- | --- | --- |
| Unit tests, lint, type checks of the glue code | Ingestion, chunking, bundle export and import, LiteLLM route config | All pass | Standard tooling |
| Scorer self-test | Recall at k, MRR, citation precision and recall with line-range overlap, key-fact matching with German stems, refusal detection, CRAG score, Wilson interval and paired bootstrap, each on hand-written cases. Every scorer also gets a wrong input it must score red, as in Keiko's memory lab | All pass | Fixtures only |
| Gold-set integrity | Every question has language, key facts, reference answer, type and must-refuse flag. Every gold citation resolves to an existing path and line range at the pinned commit of `thunderbird/thunderbird-android` or the companion repo, or to an existing issue or pull request number. Companion files carry `synthetic: true` | 100% | A file list of each pinned commit, cached |
| Citation reconciliation (Keiko `grounded-faithfulness` method) | Scripted answers with a fabricated citation, a citation outside the sent evidence, a confident answer over empty evidence, and a refusal | Detection and abstention 1.0 | Scripted answerer |
| Entailment plumbing (Keiko `grounded-entailment` method) | Per-claim support with a scripted judge: unsupported claims flagged, supported claims clean, an unavailable judge gives "unmeasured" and never "supported", a pass-through judge fails the gate | All rates 1.0, non-tautology proven | Scripted judge on the judge's call interface |
| End-to-end smoke with a scripted model | The stack under test, started with its LLM route pointed at an OpenAI-compatible stub like Keiko's `scripted-model.mjs`, answers 5 questions over a mini corpus. The stub echoes the passages it received, so the check measures ingestion, retrieval and citation plumbing | Each answer cites at least one gold ID; no fabricated citation | The stack installed on the runner; stub server |
| Model policy and egress | Every LiteLLM route declares `model_info.model_location`; a project memory with the default policy refuses a non-on-device route; evaluation runs set `RAGAS_DO_NOT_TRACK=true` and `DEEPEVAL_TELEMETRY_OPT_OUT=1` | All pass | Config files |
| Secret scan | Gitleaks over the pull request range | No finding | Pinned Gitleaks binary |

Run when ingestion, chunking or retrieval config changes (a path filter): a retrieval regression set of about 20 questions over a corpus slice of a few hundred files, with the real 0.6B embedding model and reranker on CPU through `llama-server`. Gate on recall at k and MRR equal to the committed values, because retrieval with fixed models and a fixed corpus has no sampling. If the first runs show that the scores move between identical runs, switch to a tolerance of one question. Cache the index keyed by corpus SHA, model revisions and chunker version. Measure the CPU run time first; if it exceeds 10 minutes, move this gate to the nightly run.

Every gate with a floor also gets an injected regression that must fail it (Keiko's `runRegressionProbes`). A gate that cannot fail is reported as broken.

### 4.2 Nightly on the V1 Mac (lab pair, `main` only)

The nightly run uses the full gold set of issue #11, the quick profile of issue #7, and 3 repetitions. That is about 25 minutes per repetition, so about 75 minutes. These are estimates from issue #7; the first run records real times.

| Gate | Kind | Threshold |
| --- | --- | --- |
| Fabricated citations: cited IDs outside the evidence sent to the model | Hard | 0 over all answers |
| Model location: every call went to an on-device route | Hard | 0 violations |
| Retrieval recall at k and MRR on the full corpus | Deterministic, compared with the committed baseline | Fail if more than 1 question (2 points on 50) loses its gold passage. Report the changed questions by ID |
| Per-question CRAG score (the headline of issue #7) and key-fact recall | Statistical, compared with the committed baseline | Alarm when the paired 95% interval of tonight minus baseline lies entirely below 0 and the drop is at least 5 points. Holm correction over these two metrics. An alarm starts a confirmation run of 3 more repetitions; only a confirmed drop turns the night red |
| Citation recall and precision, refusal and false-refusal rates, per-claim citation support | Reported, not gated | Means with Wilson or bootstrap intervals; a trend over 5 nights |
| Judge canaries | Hard | About 10 seeded items per night: claims paired with passages that do not support them and must be flagged, and supported claims that must pass. All 10 correct, or the judge metrics of that night are unmeasured |
| Run validity | Verdict "unmeasured" | More than 5% of answer or judge calls failed or timed out, or a model server restarted. An unmeasured night is neither pass nor fail. 3 unmeasured nights in a row raise their own alert |
| End-to-end latency p50 and p95, indexing time | Ceiling on this Mac only | Ceilings at 2 times the baseline at first; tighten once 10 nights give a spread |

Weekly, or before the bake-off and a release: the full profile of issue #7 with Ragas `Faithfulness` and `FactualCorrectness` on the first repetition (about 2.5 hours), and the SWE-QA subset. Once, and again whenever the judge model, its prompt or its quantization changes: the judge calibration against 50 hand-labelled answers and the self-consistency check of 20 answers judged twice, both from issue #7.

### 4.3 Handling judge flakiness

1. Make the judge as repeatable as the server allows. `gemma-4-31b-it-4bit` with thinking off, temperature 0, one request at a time, a fixed prompt version and a pinned model revision. Batch size changes are the main source of temperature-0 drift (Thinking Machines), and a one-request server removes them.
2. Ask for binary verdicts per claim and passage, not graded scores (Rating Roulette; Reference-Guided Verdict).
3. Cache verdicts keyed by judge model revision, prompt version, claim text and passage text, with Ragas `DiskCacheBackend` or an equivalent. Re-scoring stored answers then gives identical verdicts. The answer model samples at temperature 0.7, so new answers still get fresh judge calls.
4. If self-consistency on 20 answers judged twice is below 90%, judge each claim 3 times and take the majority. Above 90%, one call is enough.
5. Seeded canaries go through the real judge every night (section 4.2), so a drifting or broken judge invalidates its own night instead of producing a false alarm.
6. An unavailable or failed judge call never counts as "supported". It is counted, and above 5% the night is unmeasured (Keiko ADR-0144 and the MemoriaViva lab use the same rule).
7. Judge metrics never block a pull request, and they decide between stacks only after the calibration of issue #7 agrees with the hand labels.
8. Never re-run until green. A confirmation run adds repetitions and the verdict uses all of them together.

### 4.4 Baselines

- The baseline is a committed file per machine: scorecard, model revisions, LiteLLM and `mlx_lm` versions, corpus SHAs and gold-set digest, like Keiko's `*.budget.json` files and [grounded-certification-baseline.md](https://github.com/oscharko-dev/Keiko/blob/d72d6db6f2a0b97a3b5996e476bf26513395d439/docs/qa/grounded-certification-baseline.md).
- A run is compared with the baseline only when these provenance fields match, as Keiko's `compare.mjs` requires. A changed model or gold set needs a new baseline, not a comparison.
- The baseline moves only by a reviewed commit. A threshold is never lowered to make a run pass (ADR-0184 D9).
- Do not tune prompts or retrieval settings on the questions that decide the bake-off. When the gold set grows past 50, as issue #7 suggests for detecting differences under 12 points, keep about 30 questions as a holdout.

## 5. Effects on other tickets

- Issue #11 (gold set): each question needs a stable ID and pinned commit SHAs for its citations, and the set needs a mini corpus slice of about 20 questions for the pull request retrieval check, plus about 10 judge canaries.
- Issue #12 (trial run) and the start script: the script needs a non-interactive `run` command that a `launchd` job can call nightly, waits for free memory, stops everything on failure and writes the scorecard even when it fails. The CI host for model-backed checks is the V1 Mac, never a GitHub runner.
- Issue #13 (bake-off): use the same harness, the same unmeasured rule and the same judge canaries, so that bake-off numbers and nightly numbers are comparable.
- The map names `Niko4417/Keiko` as Keiko's home. That fork's `dev` stops at 2026-06-09 and lacks the 4130 branches with `lab.sh`. Current Keiko code is at `oscharko-dev/Keiko`.
