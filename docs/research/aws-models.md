# The most cost-efficient models per task on AWS

Date: 2026-11-19. Ticket: [#42](https://github.com/Niko4417/OPT-cc/issues/42). Map: [#1](https://github.com/Niko4417/OPT-cc/issues/1). Context: [ADR-0012](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0012-choose-aws-with-the-bedrock-managed-knowledge-base-as-the-one-cloud-provider.md), [ADR-0013](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0013-run-v1-on-aws-as-a-proof-of-concept-and-leave-on-device-and-client-hosted-for-later.md), [ADR-0014](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0014-require-eu-processing-for-every-third-party-cloud-route.md), [ADR-0016](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0016-pick-the-cheapest-model-per-task-whose-quality-is-good-enough.md), [ADR-0017](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0017-feed-the-managed-retrieval-arm-our-own-chunks-and-score-it-per-source-type.md), [cloud provider research](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md).

## Question

For each task of the project memory on AWS, which models are good enough at the lowest cost (ADR-0016)? The tasks are answering with citations, checking answers and finding decision conflicts between decision records, the evaluation judge, query planning and multi-turn tool calling for the agent arms, embedding (code and German and English prose), reranking, and speech to text with speaker turns. Candidates must process in the EU (ADR-0014): Bedrock serverless models in Frankfurt (eu-central-1), Bedrock EU geo inference, Bedrock Custom Model Import and SageMaker endpoints. Idle cost counts for anything billed per minute or per hour. Each candidate must be reachable through a LiteLLM route. The Managed Knowledge Base's own embedding model and reranker are not candidates.

## Answer

**Per-token Bedrock models win every language-model task, and per-call managed services win embedding, reranking and speech to text.** Nothing billed per minute or per hour comes close at the size of a small or mid-size project memory. Custom Model Import costs $8.57 per active hour for the smallest documented model size (2 Custom Model Units at $0.07144 per unit-minute in Frankfurt) [4]. A SageMaker GPU endpoint costs $0.92 to $3.13 per hour, which is $672 to $2,285 a month if it stays on [8]. The cheapest serverless answer model in the shortlist costs $1.38 a month for a small project memory. Custom Model Import also cannot load Qwen3.6-35B-A3B, because it accepts only the `Qwen3ForCausalLM` and `Qwen3MoeForCausalLM` Qwen3 architectures [5].

Two prices shape the shortlists. Claude Haiku 5.5 through EU geo inference costs $0.11 per million input tokens and $0.55 per million output tokens in the AWS Price List [3], so an answer costs less than with any other candidate except Nova Micro, Nova Lite, GLM 4.7 Flash and gpt-oss-20b. Gemma 4 26B-A4B and Gemma 4 31B cost $0.156 and $0.168 per million input tokens and lead every candidate on the FACTS Grounding leaderboard [2, 26].

Monthly costs below use the cloud provider research assumptions: 500 questions a month for a small project memory and 5,000 for a mid-size one, 20,000 input and 1,000 output tokens per answer, 30 and 200 million tokens of indexed text, 10 percent re-embedded each month, and 5 and 40 hours of recordings. Prices are list prices in USD without VAT, read from the AWS Price List API for eu-central-1 on 2026-11-19 [2, 3, 8, 12]. Idle cost is $0 for every shortlisted candidate.

### Answering with citations

| Candidate | Route | Price per million, input / output | Context | Bedrock time to first token | Evidence | Small / mid-size per month |
| --- | --- | --- | --- | --- | --- | --- |
| Claude Haiku 5.5 | EU geo | $0.11 / $0.55 | 1M | not tracked | FACTS Grounding 57.0%. Artificial Analysis index v5: 19 (low effort) to 36 (max) | $1.38 / $13.75 |
| Gemma 4 26B-A4B | in-region, `bedrock-mantle` | $0.156 / $0.48 | 256k | not tracked | FACTS 77.8% (rank 6 of 27). Vectara hallucination rate 5.2%. EuroEval German rank score 2.11 | $1.80 / $18.00 |
| Gemma 4 31B | in-region, `bedrock-mantle` | $0.168 / $0.48 | 256k | not tracked | FACTS 75.8%. Vectara 7.4%. EuroEval German 1.83 (rank 5, best candidate) | $1.92 / $19.20 |
| Qwen3 32B | in-region | $0.20 / $0.79 | 32k | 0.86 s | Vectara 5.9%. EuroEval German 2.06. No FACTS result. The 32k context leaves 12k tokens beside a 20k prompt | $2.40 / $23.95 |
| Claude Sonnet 5.5 (quality ceiling) | EU geo | $2.20 / $11.00 | 1M | 2.71 s (low effort) | FACTS 62.5%. Artificial Analysis index v5: 26 (low) to 55 (max) | $27.50 / $275.00 |

Bake-off: Gemma 4 26B-A4B, Gemma 4 31B and Claude Haiku 5.5, with Sonnet 5.5 as the reference that defines "the best shortlisted candidate" under ADR-0016. The answer-model line in the cloud provider research ($1.94 and $19.40 for Gemma 4 31B) holds.

### Checking answers and finding decision conflicts

| Candidate | Route | Price per million | Evidence | Small / mid-size per month (estimate) |
| --- | --- | --- | --- | --- |
| Claude Haiku 5.5 | EU geo | $0.11 / $0.55 | FACTS 57.0% | $0.58 / $5.78 |
| Gemma 4 31B | in-region, mantle | $0.168 / $0.48 | FACTS 75.8%, best German result | $0.79 / $7.92 |
| Qwen3 235B A22B 2507 | in-region | $0.29 / $1.16 | Vectara 9.3% (possibly the original Qwen3 235B, not 2507). No FACTS or German score | $1.45 / $14.50 |

The estimate assumes one check per answer with 8,000 input and 500 output tokens. Decision-conflict checks at ingestion add little: a few hundred pairwise checks per project memory. The checker should come from a different model family than the answer model, because language models prefer their own output [41]. With a Gemma answer model, that leaves Haiku 5.5 or Qwen3 235B as checker. No independent claim-verification benchmark (LLM-AggreFact, JudgeBench, RewardBench 2) lists any candidate [28, 29, 30].

### Evaluation judge

| Candidate | Route | Price per million | Evidence | Cost per bake-off (estimate) |
| --- | --- | --- | --- | --- |
| Claude Sonnet 5.5 | EU geo | $2.20 / $11.00 | Highest grounding and index scores of the EU-routable candidates | $22.44 |
| Claude Haiku 5.5 | EU geo | $0.11 / $0.55 | FACTS 57.0% | $1.12 |
| Gemma 4 31B | in-region, mantle | $0.168 / $0.48 | FACTS 75.8%; the lab judge in the local evaluation research | $1.50 |
| Qwen3 235B A22B 2507 | in-region | $0.29 / $1.16 | No judge benchmark | $2.78 |

The estimate assumes 100 gold questions, 4 arms and 3 repetitions, 1,200 judge calls of 6,000 input and 500 output tokens. The judge runs per bake-off, not per month, so even the most expensive candidate costs about $22 a run. No judge benchmark covers any candidate, so judge quality rests on calibration against hand labels, as the local evaluation research proposes [40].

### Query planning and multi-turn tool calling

| Candidate | Route | Price per million | Context | Evidence | Small / mid-size per month (estimate) |
| --- | --- | --- | --- | --- | --- |
| Claude Haiku 5.5 | EU geo | $0.11 / $0.55 | 1M | Artificial Analysis index v5 19 to 36, against 7 for Haiku 4.5. Not on BFCL | $2.75 / $27.50 |
| Gemma 4 31B | in-region, mantle | $0.168 / $0.48 | 256k | Artificial Analysis index v5: 4. Not on BFCL | $3.84 / $38.40 |
| Qwen3 235B A22B 2507 | in-region | $0.29 / $1.16 | 256k | BFCL v4 52.15% overall (prompt mode), 45.38% multi-turn (function calling), the highest overall BFCL score of the open models in Frankfurt | $6.96 / $69.60 |
| Claude Haiku 4.5 (evidence anchor) | EU geo | $1.10 / $5.50 | 200k | BFCL v4 68.70%, rank 6 of 108, 53.62% multi-turn | $27.50 / $275.00 |

The estimate assumes 40,000 input and 2,000 output tokens per question over several turns. Reasoning tokens are billed as output and are not included. Bake-off: Haiku 5.5, Gemma 4 31B and Qwen3 235B 2507, with Sonnet 5.5 as the ceiling. The Nova models score 1 to 2 percent on BFCL multi-turn, which looks like a harness problem but rules them out until someone checks it [24].

### Embedding

| Candidate | Route | Price | Max input, dimensions | Evidence | First run small / mid-size | Per month small / mid-size |
| --- | --- | --- | --- | --- | --- | --- |
| Cohere Embed v4 | EU geo | $0.12 per million tokens | 128k tokens, 1,536 | RTEB German1 94.96, GermanLegal1 72.85, GermanHealthcare1 85.03. Code: HumanEval 97.09, MBPP 91.21, Apps 49.66 | $3.60 / $24.00 | $0.36 / $2.40 |
| Qwen3-Embedding-0.6B | SageMaker: GPU batch job for indexing, serverless CPU endpoint for queries | instance time | 32k tokens, 1,024 | MMTEB 64.33, MTEB(Code) 75.41 (self-reported). RTEB German1 95.34, Apps 75.34 | about $0.60 to $2.40 / $4 to $16 (estimate) | under $1 / under $2 (estimate) |
| Amazon Titan Text Embeddings V2 | in-region | $0.20 per million tokens | 8,192 tokens, 1,024 | No German or code result on MTEB. AWS calls it "optimized for English" | $6.00 / $40.00 | $0.60 / $4.00 |

Bake-off: Cohere Embed v4 and Qwen3-Embedding-0.6B. Cohere v4 is stronger on German legal and health text. Qwen3-Embedding-0.6B is stronger on the one hard code set both have (Apps, 75.34 against 49.66) and runs on-device as well, so a later move off AWS keeps the same vectors (ADR-0013, embedding lock-in research). Its self-hosted figures are estimates and depend on throughput nobody has measured on AWS.

### Reranking

| Candidate | Route | Price | Evidence | Small / mid-size per month |
| --- | --- | --- | --- | --- |
| Amazon Rerank 1.0 | in-region | $1 per 1,000 search units | No independent benchmark found | $1.50 / $15.00 |
| Cohere Rerank 3.5 | in-region | $2 per 1,000 search units | Agentset ELO 1451 (Rerank 4 Pro 1629, Qwen3 Reranker 8B 1473) | $3.00 / $30.00 |

A search unit is one query with up to 100 documents [21]. The estimate assumes three reranked subqueries per question. Bake-off: both, plus a no-reranker arm. A self-hosted Qwen3-Reranker on a GPU costs $672 a month or more if it stays on, and a scale-to-zero endpoint takes minutes to start, too slow for a question [8, 9].

### Speech to text with speaker turns

| Candidate | Route | Price | German | Speakers | One-off for small / mid-size |
| --- | --- | --- | --- | --- | --- |
| Amazon Transcribe batch | Frankfurt, LiteLLM pass-through | $0.006 per minute | de-DE supported. No independent German WER found. English: 4.1% on Artificial Analysis AA-WER, 4.3% on Picovoice | up to 30 | $1.80 / $14.40 |
| Bedrock Data Automation, audio | Frankfurt | $0.006 per minute (standard) | German added 2025-11-17. No WER found | up to 30 | $1.80 / $14.40 |
| Parakeet TDT 0.6B v3 and pyannote community-1 | SageMaker async inference on ml.g4dn.xlarge, scale to zero | $0.921 per instance-hour | 4.12% German WER on the Open ASR Leaderboard | no fixed limit | about $0.50 to $1.20 / $4 to $9 (estimate) |

Bake-off: Amazon Transcribe against Parakeet with pyannote, on the project's own German workshop recordings. Transcribe is the only option with a LiteLLM route. The self-hosted pair saves at most about $10 per mid-size project memory and needs a container, a scaling policy and its own glue code.

## Bedrock in Frankfurt and the EU

The Bedrock regional availability page lists these chat models for eu-central-1 on 2026-11-19 [1]:

- In-region on `bedrock-runtime`: gpt-oss-120b, gpt-oss-20b, Qwen3 235B A22B 2507, Qwen3 32B, Qwen3-Coder-30B-A3B, MiniMax M2.1 and M2.5, Devstral 2 123B, Nemotron 3 Super 120B, GLM 4.7 Flash.
- In-region on `bedrock-mantle` only: Gemma 4 31B, Gemma 4 26B-A4B, Gemma 4 E2B.
- EU geo inference: Claude Haiku 4.5, Haiku 5.5, Sonnet 4.6, Sonnet 5, Sonnet 5.5, Opus 4.5 to 5.5, Nova 2 Lite, Nova Pro, Nova Lite, Nova Micro, Pixtral Large.
- Global only, which ADR-0014 excludes: Claude Fable and Mythos, every GPT-5.6 and GPT-6 model, Kimi K3, Grok 4.6 and 4.7, GLM 5.3.

For embedding and reranking, Frankfurt has Titan Text Embeddings V2, Cohere Embed English and Multilingual v3, Amazon Rerank 1.0 and Cohere Rerank 3.5 in-region, and Cohere Embed v4 through EU geo inference [1]. Cohere Rerank 4 is not on Bedrock [21].

Other EU regions add models in-region that Frankfurt lacks, for example Qwen3 Next 80B A3B and Voxtral in Ireland, and DeepSeek V3.2, Kimi K2.5 and GLM 5 in Stockholm and London [1]. ADR-0014 allows them, since they process in the EU, but they would put a second region into the installation. The AWS Price List for eu-central-1 carries SKUs for some of these models, such as DeepSeek V3.2 and Mistral Large 3, although the availability page does not list them for Frankfurt [1, 2]. This note trusts the availability page.

EU geo inference keeps prompts and outputs "within geographic boundaries (e.g., all EU Regions)" and is billed at the source region's price [1]. The Price List shows two Claude prices per model: "Standard" (geo or in-region) and "Standard, Global". Global is 10 percent cheaper but not allowed here [3].

## Language models

### Prices and context

| Model | Route in Frankfurt | Input / output per million | Context / max output | Answer cost per month, small / mid-size |
| --- | --- | --- | --- | --- |
| Nova Micro | EU geo | $0.046 / $0.184 | 128k / 10k | $0.55 / $5.52 |
| Nova Lite | EU geo | $0.078 / $0.312 | 300k / 10k | $0.94 / $9.36 |
| GLM 4.7 Flash | in-region | $0.08 / $0.48 | 203k / 128k | $1.04 / $10.40 |
| gpt-oss-20b | in-region | $0.09 / $0.40 | 128k / 16k | $1.10 / $11.00 |
| Claude Haiku 5.5 | EU geo | $0.11 / $0.55 | 1M / 128k | $1.38 / $13.75 |
| Gemma 4 26B-A4B | mantle | $0.156 / $0.48 | 256k / not stated | $1.80 / $18.00 |
| Gemma 4 31B | mantle | $0.168 / $0.48 | 256k / not stated | $1.92 / $19.20 |
| Nemotron 3 Super 120B | in-region | $0.18 / $0.78 | 256k / 32k | $2.19 / $21.90 |
| gpt-oss-120b | in-region | $0.20 / $0.79 | 128k / 16k | $2.40 / $23.95 |
| Qwen3 32B | in-region | $0.20 / $0.79 | 32k / 8k | $2.40 / $23.95 |
| Qwen3-Coder-30B-A3B | in-region | $0.20 / $0.79 | 256k / 16k | $2.40 / $23.95 |
| Qwen3 235B A22B 2507 | in-region | $0.29 / $1.16 | 256k / 8k | $3.48 / $34.80 |
| MiniMax M2.5 | in-region | $0.36 / $1.44 | 196k / 8k | $4.32 / $43.20 |
| Nova 2 Lite | EU geo | $0.429 / $3.597 | 1M / 64k | $6.09 / $60.89 |
| Devstral 2 123B | in-region | $0.48 / $2.40 | 256k / 32k | $6.00 / $60.00 |
| Nova Pro | EU geo | $1.05 / $4.20 | 300k / 10k | $12.60 / $126.00 |
| Claude Haiku 4.5 | EU geo | $1.10 / $5.50 | 200k / 64k | $13.75 / $137.50 |
| Claude Sonnet 5.5 | EU geo | $2.20 / $11.00 | 1M / 128k | $27.50 / $275.00 |

Prices come from the Price List offers `AmazonBedrock` and `AmazonBedrockFoundationModels` (Claude is sold through AWS Marketplace) [2, 3]. Context and max output come from the Bedrock model cards [17]. The answer cost uses 20,000 input and 1,000 output tokens per question. Haiku 5.5 bills long-context prompts at five times the standard rate ($0.55 / $2.75) [3]. The Price List does not state the threshold, and no task here sends prompts near the context limit.

### Quality evidence

No leaderboard covers every candidate, and several of the newest models appear in only one or two. All pages were read on 2026-11-19.

| Model | FACTS Grounding [26] | Vectara hallucination rate [27] | BFCL v4 overall / multi-turn [24] | EuroEval German rank score, lower is better [31] | Artificial Analysis index v5 [23] |
| --- | --- | --- | --- | --- | --- |
| Gemma 4 26B-A4B | 77.8% | 5.2% | not listed | 2.11 | not scored |
| Gemma 4 31B | 75.8% | 7.4% | not listed | 1.83 | 4 |
| Claude Sonnet 5.5 | 62.5% | not listed | not listed | not listed | 26 to 55 |
| Claude Haiku 5.5 | 57.0% | not listed | not listed | not listed | 19 to 36 |
| gpt-oss-120b | 20.1% (outlier, cause not stated) | 14.2% (served by Together AI) | not listed | 2.14 | 4 |
| Claude Haiku 4.5 | not listed | 9.8% | 68.70% / 53.62% | 2.25 | 7 |
| Qwen3 235B A22B 2507 | not listed | 9.3% (version uncertain) | 52.15% / 45.38% | incomplete | not scored |
| Qwen3 32B | not listed | 5.9% | 48.71% / 47.87% | 2.06 | not scored |
| Nova Pro | not listed | 5.1% | 24.97% / 1.88% | not listed | not scored |
| Nova 2 Lite | not listed | 5.1% | 27.10% / 2.12% | not listed | not scored |
| Nova Lite | not listed | 6.1% | not listed | not listed | not scored |
| Nova Micro | not listed | 5.5% | 22.29% / 1.38% | not listed | not scored |
| MiniMax M2.5 | not listed | 9.1% | not listed | not listed | not scored |
| GLM 4.7 Flash | not listed | 9.3% | not listed | 2.28 (third-party quantization) | not scored |
| gpt-oss-20b | not listed | not listed | not listed | 2.35 (medium effort) | not scored |

Notes on the sources:

- FACTS Grounding (version 3, updated 2026-11-14) scores whether a long-context answer stays within the supplied document. It is the closest public benchmark to answering with citations. 27 models are listed.
- The Vectara leaderboard (HHEM-2.3, updated 2026-10-22) measures hallucinations in English summaries. Scores between 5% and 10% are close together.
- BFCL v4 was last updated 2026-04-12 and lists 108 configurations. It predates Haiku 5.5, Sonnet 5.5 and Gemma 4.
- EuroEval's German leaderboard (version 18.4.0, data from 2026-11-18) ranks German language understanding and generation. Its leader, GPT-6 Astra, scores 1.39. Gemma 4 31B at 1.83 is the best of the candidates.
- Artificial Analysis Intelligence Index v5 uses a strict scale; its top model scores 62. It gives a range for models with several reasoning efforts.
- tau2-bench, LLM-AggreFact, JudgeBench and RewardBench 2 list none of the candidates [25, 28, 29, 30].
- The two outliers, Nova near zero on BFCL multi-turn and gpt-oss-120b at 20.1% on FACTS, may be harness or filter effects. Neither source explains them.

The Gemma 4 models lead on grounding and German, and are cheap. Claude Haiku 5.5 costs less still, and its agentic index is far higher than Gemma 4's, but it has only one independent grounding result and no German result. That is why both families go into the bake-off.

### Latency

Artificial Analysis publishes "time to first answer token" for Amazon Bedrock, averaged over five days, but not in which region it measures [23]. Without reasoning: Qwen3 32B 0.86 s, Claude Haiku 4.5 0.93 s, Nova Micro 1.26 s, Qwen3-Coder-30B 1.54 s, Nova 2 Lite 1.58 s, Nova Lite 1.61 s, Qwen3 235B 2507 1.77 s, GLM 4.7 Flash 1.87 s, gpt-oss-120b (low effort) 2.24 s, Nova Pro 2.65 s, Sonnet 5.5 (low effort) 2.71 s. With reasoning, the figure includes thinking time and reaches 13 to 254 seconds. Artificial Analysis does not track Haiku 5.5, Gemma 4, MiniMax M2.5, Nemotron 3 Super or Devstral 2 on Bedrock, so the bake-off must measure their latency in Frankfurt.

### Why per-minute and per-hour hosting loses

**Custom Model Import.** It runs in eu-central-1 [5]. Billing is per Custom Model Unit per minute, $0.07144 in Frankfurt, in 5-minute windows from the first call, plus $1.95 per unit per month for storage [4, 7]. AWS's examples: a Llama 3.1 8B model with 128k context needs 2 units, and a 70B model needs 8 [4]. A copy scales to zero after 5 minutes without calls, and the next call waits "tens of seconds" [4]. One active hour therefore costs $8.57 at 2 units and $34.29 at 8. If consultants use the project memory 40 hours a month (small, estimate) or 170 hours (mid-size, estimate), a 2-unit model costs about $343 or $1,457 a month, against $1.92 or $19.20 for Gemma 4 31B on demand. Other limits [5, 6]:

- Supported Qwen architectures are Qwen2, Qwen2.5, Qwen2-VL, Qwen2.5-VL, and for Qwen3 only `Qwen3ForCausalLM` and `Qwen3MoeForCausalLM`, with transformers 4.51.3. Qwen3.6-35B-A3B declares `Qwen3_5MoeForConditionalGeneration` (memory fit research) and is not supported. Gemma is not in the list.
- Context must be under 128k tokens. Embedding models are not supported.
- Tool calling is documented only for GPT-OSS imports, and GPT-OSS imports run only in us-east-1.

Custom Model Import therefore offers nothing for the agent arms in Frankfurt and nothing cheaper for answering.

**SageMaker real-time endpoints.** Frankfurt hosting prices per hour are ml.g4dn.xlarge (T4, 16 GB) $0.921, ml.g6.xlarge (L4, 24 GB) $1.409, ml.g5.xlarge (A10G, 24 GB) $1.761 and ml.g6e.xlarge (L40S, 48 GB) $3.13 [8]. Qwen3.6-35B-A3B in 8-bit needs about 35 GB of weights and so an ml.g6e.xlarge (estimate): $2,285 a month always on, or $592 for 9 hours on 21 working days. Endpoints with inference components can scale to zero, but "the provisioning process takes several minutes" and calls fail until it ends [9]. That is acceptable for batch work, not for a question.

**SageMaker Serverless Inference.** It scales to zero and bills per second of use, $0.00008 per second at 4 GB in Frankfurt, but it has no GPUs and at most 6 GB of memory [8, 10]. It can run a 0.6B embedding model for short queries. It cannot run a chat model of useful size.

## Embedding

The Managed Knowledge Base arm embeds with its own model (ADR-0017). The choice here applies to the self-built arms.

| Model | Max input | Dimensions | Frankfurt route | Price per million tokens | License |
| --- | --- | --- | --- | --- | --- |
| Cohere Embed v4 | 128k tokens | 1,536 | EU geo, routes to Frankfurt, Stockholm, Milan, Zurich, Dublin, Paris | $0.12 | proprietary |
| Cohere Embed Multilingual v3 | 512 tokens | 1,024 | in-region | $0.10 | proprietary |
| Amazon Titan Text Embeddings V2 | 8,192 tokens or 50,000 characters | 1,024, 512, 256 | in-region | $0.20 | proprietary |
| Qwen3-Embedding-0.6B | 32k tokens | 1,024 | SageMaker | instance time | Apache-2.0 |
| Qwen3-Embedding-4B | 32k tokens | 2,560 | SageMaker GPU | instance time | Apache-2.0 |

Sources: [3, 18, 19, 20, 32]. Titan V2 costs $0.02 per million in us-east-1 and ten times that in Frankfurt [2]. The embedding lock-in research used the same Frankfurt price.

Retrieval quality, nDCG@10 times 100, from the MTEB results repository [33] and the Qwen3 Embedding paper [32]:

| Task | Cohere Embed v4 | Qwen3-Emb 0.6B | Qwen3-Emb 4B | Cohere Multilingual v3 |
| --- | --- | --- | --- | --- |
| RTEB German1 | 94.96 | 95.34 | 96.52 | 91.90 |
| RTEB GermanLegal1 | 72.85 | 61.80 | 70.47 | 65.17 |
| RTEB GermanHealthcare1 | 85.03 | 71.35 | 84.80 | 72.75 |
| AppsRetrieval (code) | 49.66 | 75.34 | 89.18 | 31.91 |
| HumanEvalRetrieval (code) | 97.09 | not run | 98.12 | 81.56 |
| MBPPRetrieval (code) | 91.21 | not run | 91.44 | 78.02 |
| DS1000Retrieval (code) | 63.84 | not run | 63.62 | 50.21 |
| MMTEB mean (self-reported) | not run | 64.33 | 69.45 | 61.12 |
| MTEB(Code) mean (self-reported) | not run | 75.41 | 80.06 | 51.94 |

Titan V2 has results only for English MTEB v1 tasks and ChemTEB, submitted rather than run by MTEB, and none for German or code [33]. AWS documents it as "optimized for English" and warns that cross-language queries "will return sub-optimal results" [18]. It is the most expensive of the three managed options and has no evidence for two of the three content types, so it is off the shortlist. Cohere Multilingual v3 truncates at 512 tokens, below the size of a typical code chunk, so it is off as well.

**Self-hosting Qwen3-Embedding-0.6B.** The first embedding run and the monthly 10 percent run go to a SageMaker job on an ml.g6.xlarge at $1.409 an hour. At 5,000 to 20,000 tokens a second on one L4 GPU (estimate, not measured), 30 million tokens take 25 to 100 minutes and 200 million take 3 to 11 hours. Query embeddings go to a serverless endpoint with 4 GB, where a query of a few dozen tokens takes well under a second (estimate), so 2,000 to 20,000 queries a month cost cents. The serverless endpoint has no idle cost but a cold start after idle periods, which the bake-off must measure [10]. The 4B and 8B models need more than the 6 GB serverless limit, so they need a GPU that stays on ($672 a month or more) or accepts minutes of start-up [8, 9, 10].

## Reranking

- Amazon Rerank 1.0 and Cohere Rerank 3.5 run in-region in Frankfurt with no geo profile [21, 22].
- The Rerank API takes one query and 1 to 1,000 documents per call. One search unit covers up to 100 documents [21].
- Prices in Frankfurt: Amazon Rerank $1 per 1,000 search units in the `AmazonBedrock` offer, Cohere Rerank 3.5 $0.002 per search unit in the Marketplace offer [2, 3].
- Cohere Rerank 3.5 has a 4k-token context window per document [22]. AWS gives no per-document limit for Amazon Rerank.

Quality evidence is thin. The Agentset reranker leaderboard (updated 2026-02-15, ELO from a GPT-5 judge over FiQA, SciFact and PG) ranks Cohere Rerank 4 Pro at 1629, Cohere Rerank 4 Fast 1510, Qwen3 Reranker 8B 1473 and Cohere Rerank 3.5 1451. It does not list Amazon Rerank [34]. The Qwen3 paper reports Qwen3-Reranker-0.6B at 73.42 on MTEB-Code reranking and 66.36 on MMTEB-R, against 41.38 and 58.36 for bge-reranker-v2-m3 (self-reported) [32]. I found no independent result for Amazon Rerank 1.0 and no German reranking benchmark.

## Speech to text

- **Amazon Transcribe.** Batch transcription supports de-DE and en-US. Speaker diarization labels up to 30 speakers at no extra charge [13]. The Frankfurt price is $0.0001 per second, $0.36 per audio hour, billed per second [12]. Artificial Analysis measures 4.1% English WER (AA-WER v2.2), level with Whisper large-v3 and ahead of Parakeet v3 at 4.5% [35]. Picovoice's benchmark gives 4.3% English WER against 5.7% for Whisper large-v3 [36]. Picovoice also runs German sets but publishes German results only as charts.
- **Bedrock Data Automation, audio.** It runs in Frankfurt at $0.006 a minute (standard) or $0.009 (custom), labels up to 30 speakers, and added German on 2025-11-17 [2, 14]. No WER is published.
- **Parakeet TDT 0.6B v3 with pyannote community-1.** This is the on-device pair from the local transcription research: 4.12% German WER on the Open ASR Leaderboard and a DER of 17.0% on AMI headset audio [37, 38]. On AWS it needs an NVIDIA GPU. SageMaker asynchronous inference accepts payloads up to 1 GB, processes for up to an hour and can scale to zero [11]. With an ml.g4dn.xlarge at $0.921 an hour, the cost per audio hour is about $0.09 to $0.23, including instance start-up (estimate).
- **Not suitable.** Voxtral Small and Mini run through the chat API in Ireland, London and Milan, document no diarization or timestamps, and Voxtral Small is in legacy status with end of life on 2027-03-30 [15]. Nova 2 Sonic is a speech-to-speech model for bidirectional streams, offered in the EU only in Stockholm [16].

## LiteLLM routes

| Candidate | LiteLLM route | Status |
| --- | --- | --- |
| Bedrock runtime models (gpt-oss, Qwen3, Nemotron, GLM, MiniMax) | `bedrock/converse/<model id>` | Documented [39] |
| Claude and Nova through EU geo | `bedrock/eu.<model id>` | Documented [39] |
| Gemma 4 on `bedrock-mantle` | `bedrock_mantle/` provider, or `openai/` with the mantle `api_base` | Provider in the LiteLLM repo; the `openai/` path is documented by AWS (cloud provider research, A2) |
| Titan V2 and Cohere Embed v4 | `bedrock/amazon.titan-embed-text-v2:0`, `bedrock/eu.cohere.embed-v4:0` | Titan documented; Cohere v4 only in the model catalog and code, not in the docs page [39] |
| Amazon Rerank 1.0, Cohere Rerank 3.5 | `bedrock/arn:aws:bedrock:eu-central-1::foundation-model/<model id>` on `/rerank` | Documented [39] |
| Custom Model Import | `bedrock/qwen3/<model ARN>`, `bedrock/llama/<model ARN>` | Documented [39] |
| SageMaker chat and embeddings | `sagemaker_chat/<endpoint>`, `sagemaker/<endpoint>` | Documented. LiteLLM's support matrix marks SageMaker embeddings unsupported, which contradicts its docs. Asynchronous endpoints are not covered [39] |
| Amazon Transcribe | Proxy pass-through `POST /transcribe/{operation}`, with SigV4 signing, per-key job ownership and an S3 bucket allowlist | In the LiteLLM repo; not an `/audio/transcriptions` provider [39] |
| Bedrock Data Automation | none found | Not checked further |

Every route declares `model_info.model_location: third-party-cloud`, as the model server research proposes. The Parakeet and pyannote pair on SageMaker asynchronous inference has no LiteLLM route, so the ingestion job would call it directly, outside the model policy check.

## Open points

- Confirm Claude Haiku 5.5's EU geo price of $0.11 and $0.55 per million tokens in the console or a test bill. The Price List API is the only source, and the pricing page shows no Claude 5.5 rows.
- Check whether Gemma 4 on `bedrock-mantle` supports tool calls in the OpenAI chat format, and measure the Gemma 4 and Haiku 5.5 latency in Frankfurt.
- Find out why the Nova models score near zero on BFCL multi-turn and gpt-oss-120b scores 20.1% on FACTS Grounding.
- Measure German WER and diarization error on the project's own recordings for Transcribe and the Parakeet and pyannote pair. No independent German numbers exist for Transcribe.
- Measure Qwen3-Embedding-0.6B throughput on an ml.g6.xlarge and cold start on a serverless endpoint.
- Decide how the speech-to-text step reaches LiteLLM: the Transcribe pass-through works through the proxy, but no standard transcription route covers Transcribe or a SageMaker asynchronous endpoint.
- Judge quality for every candidate rests on calibration against hand labels, because no judge benchmark lists them.

## Method

I read the Bedrock regional availability page on 2026-11-19 and parsed its per-model tables for eu-central-1 and the other EU regions. Prices come from the AWS Price List API for eu-central-1 (offers `AmazonBedrock`, `AmazonBedrockFoundationModels`, `AmazonSageMaker` and `transcribe`), read the same day, and from the Bedrock pricing page for Custom Model Import. Quality numbers come from the leaderboards and papers named in each table, read on 2026-11-19, with self-reported vendor numbers marked as such. LiteLLM support comes from its documentation and its GitHub repository on 2026-11-19. Monthly costs reuse the assumptions of the cloud provider research. Token counts for checking, judging and tool calling are my estimates, marked as such. I did not deploy anything or run any model.

## Sources

1. [Regional availability by models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-region-compatibility.html), Amazon Bedrock user guide, read 2026-11-19
2. [AWS Price List API, AmazonBedrock, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonBedrock/current/eu-central-1/index.json), read 2026-11-19
3. [AWS Price List API, AmazonBedrockFoundationModels, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonBedrockFoundationModels/current/eu-central-1/index.json), read 2026-11-19
4. [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), Custom Model Import section, read 2026-11-19
5. [Use Custom model import](https://docs.aws.amazon.com/bedrock/latest/userguide/model-customization-import-model.html), supported architectures and regions
6. [Invoke your imported model](https://docs.aws.amazon.com/bedrock/latest/userguide/invoke-imported-model.html)
7. [Calculate the cost of running a custom model](https://docs.aws.amazon.com/bedrock/latest/userguide/import-model-calculate-cost.html)
8. [AWS Price List API, AmazonSageMaker, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonSageMaker/current/eu-central-1/index.json), read 2026-11-19
9. [Scale an endpoint to zero instances](https://docs.aws.amazon.com/sagemaker/latest/dg/endpoint-auto-scaling-zero-instances.html)
10. [Deploy models with Amazon SageMaker Serverless Inference](https://docs.aws.amazon.com/sagemaker/latest/dg/serverless-endpoints.html)
11. [Asynchronous inference](https://docs.aws.amazon.com/sagemaker/latest/dg/async-inference.html) and [Autoscale an asynchronous endpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/async-inference-autoscale.html)
12. [AWS Price List API, transcribe, eu-central-1](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/transcribe/current/eu-central-1/index.json) and [Amazon Transcribe pricing](https://aws.amazon.com/transcribe/pricing/), read 2026-11-19
13. [Supported languages](https://docs.aws.amazon.com/transcribe/latest/dg/supported-languages.html) and [Partitioning speakers (diarization)](https://docs.aws.amazon.com/transcribe/latest/dg/diarization.html), Amazon Transcribe
14. [SpeakerLabelingConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_data-automation_SpeakerLabelingConfiguration.html) and [Bedrock Data Automation adds 10 languages](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-bedrock-data-automation-10-languages), 2025-11-17
15. [Voxtral Small 24B 2507 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-mistral-ai-voxtral-small-24b-2507.html)
16. [Amazon Nova 2 Sonic model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-amazon-nova-2-sonic.html)
17. [Bedrock model cards](https://docs.aws.amazon.com/bedrock/latest/userguide/model-cards.html), one page per model, read 2026-11-19
18. [Amazon Titan Text Embeddings models](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)
19. [Cohere Embed v4 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-cohere-embed-v4.html)
20. [Cohere Embed Multilingual model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-cohere-embed-multilingual.html)
21. [Supported Regions and models for reranking](https://docs.aws.amazon.com/bedrock/latest/userguide/rerank-supported.html), [Rerank API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_Rerank.html) and [Rerank pricing](https://docs.aws.amazon.com/bedrock/latest/userguide/rerank-pricing.html)
22. [Cohere Rerank 3.5 model card](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-cohere-rerank-3-5.html)
23. [Artificial Analysis, Amazon Bedrock provider](https://artificialanalysis.ai/providers/amazon_bedrock) and model pages, Intelligence Index v5.0.1, read 2026-11-19
24. [Berkeley Function Calling Leaderboard v4](https://gorilla.cs.berkeley.edu/leaderboard.html), last updated 2026-04-12
25. [tau2-bench leaderboard](https://taubench.com), read 2026-11-19
26. [FACTS Grounding leaderboard](https://www.kaggle.com/benchmarks/google/facts-grounding), Google DeepMind on Kaggle, version 3, updated 2026-11-14
27. [Vectara hallucination leaderboard](https://github.com/vectara/hallucination-leaderboard), HHEM-2.3, updated 2026-10-22
28. [LLM-AggreFact leaderboard](https://llm-aggrefact.github.io/), read 2026-11-19
29. [JudgeBench](https://huggingface.co/spaces/ScalerLab/JudgeBench), read 2026-11-19
30. [RewardBench 2 results](https://huggingface.co/datasets/allenai/reward-bench-2-results), read 2026-11-19
31. [EuroEval German leaderboard](https://euroeval.com/leaderboards/german), version 18.4.0, data from 2026-11-18
32. Zhang et al., [Qwen3 Embedding: Advancing Text Embedding and Reranking Through Foundation Models](https://arxiv.org/abs/2506.05176), arXiv 2506.05176
33. [MTEB results repository](https://github.com/embeddings-benchmark/results), folders `amazon__Titan-text-embeddings-v2` and `Cohere__Cohere-embed-v4.0`, read 2026-11-19
34. [Agentset reranker leaderboard](https://agentset.ai/rerankers), updated 2026-02-15
35. [Artificial Analysis speech-to-text](https://artificialanalysis.ai/speech-to-text), AA-WER v2.2, read 2026-11-19
36. [Picovoice speech-to-text benchmark](https://github.com/Picovoice/speech-to-text-benchmark), read 2026-11-19
37. [Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard), via the [local transcription research](https://github.com/Niko4417/OPT-cc/blob/research/local-transcription/docs/research/local-transcription.md)
38. [nvidia/parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) and [pyannote/speaker-diarization-community-1](https://huggingface.co/pyannote/speaker-diarization-community-1) model cards
39. LiteLLM: [Bedrock](https://docs.litellm.ai/docs/providers/bedrock), [Bedrock rerank](https://docs.litellm.ai/docs/providers/bedrock_rerank), [Bedrock imported models](https://docs.litellm.ai/docs/providers/bedrock_imported), [SageMaker](https://docs.litellm.ai/docs/providers/aws_sagemaker), [provider_endpoints_support.json](https://github.com/BerriAI/litellm/blob/main/provider_endpoints_support.json), [model_prices_and_context_window.json](https://github.com/BerriAI/litellm/blob/main/model_prices_and_context_window.json) and [llm_passthrough_endpoints.py](https://github.com/BerriAI/litellm/blob/main/litellm/proxy/pass_through_endpoints/llm_passthrough_endpoints.py), read 2026-11-19
40. Project research: [cloud provider](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md), [local evaluation](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md), [embedding lock-in](https://github.com/Niko4417/OPT-cc/blob/research/embedding-lock-in/docs/research/embedding-lock-in.md), [memory fit](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md), [model server](https://github.com/Niko4417/OPT-cc/blob/research/model-server/docs/research/model-server.md)
41. Panickssery et al., [LLM Evaluators Recognize and Favor Their Own Generations](https://arxiv.org/abs/2404.13076), arXiv 2404.13076
