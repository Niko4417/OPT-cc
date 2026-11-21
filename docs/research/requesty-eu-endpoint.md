# Requesty's EU endpoint next to the Bedrock knowledge base

Date: 2026-11-21. Ticket: [Find whether Requesty's EU endpoint can serve the models next to the Bedrock knowledge base](https://github.com/Niko4417/OPT-cc/issues/55). Map: [the wayfinder map](https://github.com/Niko4417/OPT-cc/issues/1). Context: [ADR-0012](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0012-choose-aws-with-the-bedrock-managed-knowledge-base-as-the-one-cloud-provider.md), [ADR-0013](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0013-run-v1-on-aws-as-a-proof-of-concept-and-leave-on-device-and-client-hosted-for-later.md), [ADR-0014](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0014-require-eu-processing-for-every-third-party-cloud-route.md), [ADR-0016](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0016-pick-the-cheapest-model-per-task-whose-quality-is-good-enough.md), [ADR-0017](https://github.com/Niko4417/OPT-cc/blob/main/docs/adr/0017-feed-the-managed-retrieval-arm-our-own-chunks-and-score-it-per-source-type.md), [AWS models research](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md), [cloud provider research](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md), [AWS installation research](https://github.com/Niko4417/OPT-cc/blob/research/aws-installation/docs/research/aws-installation.md), [model server research](https://github.com/Niko4417/OPT-cc/blob/research/model-server/docs/research/model-server.md).

## Question

Can the installation keep the AWS retrieval services, above all the Bedrock Managed Knowledge Base, and run its models over Requesty's EU endpoint? Or do the AWS retrieval services tie the installation to Bedrock models? Is LiteLLM still needed if Bedrock is used? The ticket asks seven points: the EU endpoint and its data handling, model coverage per task, price, which retrieval steps force a Bedrock model, lock-in, LiteLLM, and the bake-off.

## Answer

**The installation can keep the Bedrock Managed Knowledge Base and run every model call it controls over Requesty, but the knowledge base's own steps stay on AWS models.** The knowledge base has a retrieve-only call, `Retrieve`, and its agentic call, `AgenticRetrieveStream`, can return results without an answer (`generateResponse: false`). So the answer, the check, the judge and the self-built arms' planning can run on any model behind LiteLLM, including Requesty routes [A4, A5, A6, A7]. Four steps that the knowledge base runs itself can only use AWS models:

- Parsing uses the managed parser, fixed by AWS [A3].
- Embedding uses the managed model, fixed by AWS, or one of five Bedrock embedding models chosen at creation and never changed [A2].
- Reranking uses the managed reranker, fixed by AWS, or a Bedrock reranker. It can also be switched off, and then the app can rerank the returned chunks with any model [A4, A5].
- Agentic planning, and the optional answer of `AgenticRetrieveStream`, uses a managed model fixed by AWS or a Bedrock model, including `bedrock-mantle` models [A6, A7].

No knowledge base API takes a model outside Bedrock. This is lock-in for the managed arm, not for the installation. The self-built arms and the app's own model calls stay free to use Requesty.

**Requesty's EU endpoint meets ADR-0014 only with three settings, and it adds little that LiteLLM lacks.** The endpoint `https://router.eu.requesty.ai/v1` runs in Frankfurt on AWS eu-central-1, but it only governs Requesty's own processing. EU inference needs the organisation's model hosting location restricted to EU, the serving region restricted to EU, and EU-only approved models [R1, R2]. Prompt and output logging is on by default and kept 30 days in the EU, so zero data retention must also be switched on [R3, R7, R9]. The legal entity is Requesty Ltd in London [R8], which is not an EU entity but is covered by the EU adequacy decision for the UK, renewed on 2025-12-19 [E1]. Requesty has no rerank endpoint, no Bedrock bring-your-own-key option, and no native LiteLLM provider [R6, R12, L1]. It cannot replace LiteLLM, because the app needs the per-route `model_location`, the LiteLLM budget, and routes that Requesty cannot reach: Bedrock rerank, Amazon Transcribe and later on-device or client-hosted models.

**LiteLLM stays needed even with Bedrock only.** It carries the `model_info.model_location` key per route, the budget that stops spend at once, one API for the app, and the self-built arms' independence from AWS that ADR-0013 keeps for a later move (model server research, AWS installation research).

Three numbers matter most:

1. Requesty adds 5 percent to model cost. It charges the fee on every top-up of a prepaid balance, and its catalog prices for the Bedrock models I checked equal the AWS list prices [R4, R5, R11]. Claude Haiku 5.5 in eu-central-1 costs $0.11 / $0.55 per million tokens in both, so $0.1155 / $0.5775 effective over Requesty.
2. Qwen3-Embedding-8B on Nebius in the EU costs $0.01 per million tokens over Requesty [R11]. A first embedding run of a mid-size project memory (200 million tokens) costs about $2.10 with the fee, against $24.00 for Cohere Embed v4 on Bedrock (AWS models research). It is open-weight, so the same vectors work on-device later.
3. Of the 856 models in Requesty's public catalog, 254 carry the hosting location `eu` [R11]. None of them is Qwen3.6-35B-A3B or Gemma 4, and none is a reranker. Bedrock in Frankfurt has Gemma 4 and two rerankers.

## 1. The EU endpoint

**URL and location.** Requesty documents `https://router.eu.requesty.ai/v1` for OpenAI-compatible requests and `https://router.eu.requesty.ai` for Anthropic-compatible requests. It states that the endpoint runs in "Frankfurt, Germany (AWS `eu-central-1`)" and that "All processing and storage by Requesty stays in the EU" [R1]. The sub-processor list names one infrastructure sub-processor, Amazon Web Services EMEA SARL, in Frankfurt [R8].

**Two layers.** Requesty says that using the EU endpoint "alone only guarantees that Requesty's processing stays in the EU" [R1]. A request for a global model, such as `anthropic/claude-sonnet-5-5`, "will be forwarded outside the EU for inference" [R1]. By default the EU endpoint "can route to any model, including models hosted outside the EU" [R1].

**Settings that pin a request to the EU.** An organisation admin sets these on the Compliance page [R1, R2]:

- Serving region restricted to EU. Requests to the global, US or AP endpoint are then rejected.
- Model hosting location restricted to EU. Requesty then rejects models hosted elsewhere, "including globally hosted (multi-region) models and models with no recorded location", even if they are approved. The rule covers chat, embedding, image, speech, transcription and decision models.
- Approved models limited to EU models. A request for any other model is rejected.
- Zero data retention, on the Requesty side and, separately, a requirement that providers retain nothing. Both take effect at once, and only Requesty support can turn them off again.

**What Requesty keeps.** Requesty always stores telemetry: token counts, cost, latency, model and provider used, status, timestamps, API key ID, tags and deployment region [R3]. It stores prompts and responses only if the API key has logging on and the organisation has no zero data retention [R3]. The DPA page states that "prompt and output logging is on by default on self serve plans with a 30 day retention window" [R7]. The privacy policy says logged data "is stored encrypted within the EU for up to 30 days" [R9]. Logged payloads include a "Full request body stored in object storage" [R3]. Requesty's caching is the provider's own prompt caching. Requesty adds cache breakpoints and says that auto caching "is not server-side response storage" [R16].

**Legal entity and contract.** The operator is Requesty Ltd, 71-75 Shelton Street, Covent Garden, London WC2H 9JQ, UK [R8]. The DPA covers GDPR Article 28, is pre-signed by Requesty on the Compliance page, and covers Requesty only, "not the upstream model providers" [R2, R7]. The sub-processor list (last updated 2026-10-29) promises 30 days' notice of changes [R8]. The European Commission renewed the UK adequacy decision under the GDPR on 2025-12-19, valid until 2031-12-27 [E1]. ADR-0014 asks where data is processed and kept, which is Frankfurt. The UK seat is a scored criterion, not a hard rule.

**Upstream providers and regions.** The EU models in the catalog come from twelve providers [R11]. The sub-processor list gives each one's location [R8]:

| Provider in the catalog | Location per sub-processor list | Examples |
| --- | --- | --- |
| `bedrock` | AWS Bedrock, US and EU | Claude 4.5 to 5.5 in eu-central-1, eu-west-1, eu-west-3 and eu-north-1; MiniMax M2.5; Kimi K2.5 in eu-north-1 and eu-west-2 |
| `vertex` | Google Vertex AI, US and EU | Claude via `@eu`, Gemini 2.5 to 4, Gemini Embedding 2 |
| `azure` | Microsoft Azure AI, US and EU | GPT-4.1, GPT-5.x and GPT-6 in France Central, Sweden Central, Germany West Central and West Europe; text-embedding-3 in France Central |
| `mistral` | Mistral AI SAS, EU | Mistral Small, Medium and Large 4, Codestral, Voxtral |
| `nebius` | Nebius AI, EU | Qwen3 235B 2507, Qwen3 30B-A3B 2507, gpt-oss-120b, Gemma 3 27B, Qwen3-Embedding-8B |
| `scaleway` | Scaleway SAS, France, zero data retention | gpt-oss-120b, GLM 5.2, Whisper large-v3 |
| `lyceum`, `melious`, `sference`, `tensorx`, `inceptron` | Germany; Germany, Finland and Ireland; Finland; Ireland; Sweden | DeepSeek V4, GLM 5.3, Kimi K3, MiniMax M3, Qwen3.8 |
| `coding` | Not listed. The region suffixes are Google EU regions | Gemini 2.5 |

Bedrock over Requesty runs in Requesty's AWS account, not ours. Requesty's bring-your-own-key list has OpenAI, Anthropic, Google AI Studio, Vertex AI, Mistral and xAI, and no AWS Bedrock [R6]. The data protection terms are then AWS's terms with Requesty, which our DPA with Requesty does not cover [R2]. I could not find out whether Requesty's `@eu-central-1` Claude routes call Bedrock in-region or through the EU geo inference profile. The AWS models research found that Claude Haiku 5.5 in Frankfurt runs only through EU geo inference, so the likely answer is EU geo, which processes anywhere in the EU (inferred).

**Catalog oddities.** `vertex/google/gemini-embedding-001@europe-west1` and its sister routes carry an EU region in the name but `geolocation: global` in the catalog [R11]. With the hosting location restricted to EU, Requesty would reject them. The catalog is the source of truth for the restriction, so a mislabelled route fails closed.

**Limits and reliability.** Requesty does not cap requests per minute. It caps requests in flight at the prepaid balance times a tier multiplier of 5, 10 or 15. A $20 balance on tier 1 allows 100 requests in flight [R13]. A request beyond the limit gets HTTP 429, and a request with an empty balance gets HTTP 402 [R4, R13]. Uploads to the transcription endpoint are limited to 32 MB [R18]. Requesty publishes no uptime figure for pay-as-you-go plans. Custom SLAs are an Enterprise feature [R5]. Its security page states "SOC 2 Type II programme in progress" (status September 2026) and points to trust.requesty.ai, whose content I could not read [R10].

**Verdict for ADR-0014.** A Requesty route processes and keeps data in the EU if all of the following hold: the EU endpoint, the serving region restricted to EU, the hosting location restricted to EU, EU-only approved models, and zero data retention on (or logging off on the key, which keeps logs in the EU for 30 days). Each of these rests on Requesty's own statements. I found no audit report that confirms them.

## 2. Coverage per task

The EU column lists models with `geolocation: eu` in the public catalog of `https://router.eu.requesty.ai/v1/models/all`, read without a key on 2026-11-21 [R11]. The documentation says that an unauthenticated call returns "All publicly available models" [R11]. An organisation's own list can only be shorter, because approved models narrow it. Prices are list prices per million tokens before Requesty's 5 percent fee. Bedrock prices come from the AWS models research.

| Task | Requesty EU models (input / output per million) | Bedrock Frankfurt shortlist | Only on Requesty EU | Only on Bedrock Frankfurt |
| --- | --- | --- | --- | --- |
| Answering with citations | Claude Haiku 5.5 $0.11 / $0.55; gpt-oss-120b (Nebius) $0.15 / $0.60; Qwen3 30B-A3B 2507 (Nebius) $0.10 / $0.30; GPT-5 nano (Azure) $0.055 / $0.44; GPT-6 Luna (Azure) $0.12 / $0.60; Mistral Small $0.165 / $0.66; Claude Sonnet 5.5 $2.20 / $11.00 | Claude Haiku 5.5, Gemma 4 26B-A4B, Gemma 4 31B, Qwen3 32B, Sonnet 5.5 as ceiling | Qwen3 30B-A3B 2507, Mistral, GPT on Azure, Gemini on Vertex | Gemma 4 26B-A4B and 31B, Qwen3 32B, Nova |
| Checking and decision conflicts | Claude Haiku 5.5; Qwen3 235B A22B 2507 (Nebius) $0.20 / $0.60 | Haiku 5.5, Gemma 4 31B, Qwen3 235B 2507 at $0.29 / $1.16 | Same as above | Gemma 4 31B |
| Evaluation judge | Claude Sonnet 5.5 $2.20 / $11.00; Haiku 5.5; Qwen3 235B 2507 | Sonnet 5.5, Haiku 5.5, Gemma 4 31B, Qwen3 235B 2507 | GPT-6 Sol (Azure), Gemini 4 (Vertex) | Gemma 4 31B |
| Query planning and tool calling | Claude Haiku 5.5; Qwen3 235B 2507 $0.20 / $0.60; gpt-oss-120b $0.15 / $0.60 | Haiku 5.5, Gemma 4 31B, Qwen3 235B 2507 | Same as above | Gemma 4 31B |
| Embedding | Qwen3-Embedding-8B (Nebius) $0.01; text-embedding-3-small (Azure France) $0.02; text-embedding-3-large (Azure France) $0.13; Gemini Embedding 2 (Vertex EU) $0.22 | Cohere Embed v4 $0.12, Qwen3-Embedding-0.6B on SageMaker | Qwen3-Embedding-8B, OpenAI and Gemini embeddings | Cohere Embed v4, Titan V2 |
| Reranking | None. Requesty has no rerank endpoint [R12] | Amazon Rerank 1.0 $1, Cohere Rerank 3.5 $2 per 1,000 search units | None | Both rerankers |
| Speech to text with speaker turns | Voxtral Mini Transcribe 3.0 $0.0035 per minute; Voxtral Small Transcribe 3.0 $0.0040; Whisper large-v3 (Scaleway) $0.0036. No speaker labels documented [R18] | Amazon Transcribe $0.006 per minute with up to 30 speakers | Voxtral, Whisper | Speaker diarization |

Notes:

- **Qwen3.6-35B-A3B is on neither.** The whole Requesty catalog has only `alibaba/qwen3.6-plus`, hosted globally by Alibaba Cloud, whose sub-processor entry is Singapore [R8, R11]. Requesty EU therefore does not bring back the lab answer model.
- **Gemma 4 is not in the EU on Requesty.** DeepInfra, Novita, Parasail and Sail serve Gemma 4 26B-A4B and 31B, all with `geolocation: global` and US sub-processor entries [R8, R11]. The two Gemma 4 models lead the FACTS Grounding results in the AWS models research, and only Bedrock serves them in the EU.
- **Bedrock models over Requesty** are Claude (Haiku 4.5 and 5.5, Sonnet 4.5 to 5.5, Opus 4.5 to 5.5), MiniMax M2.5 and Kimi K2.5 [R11]. Gemma 4, gpt-oss, Qwen3, Nova and every Bedrock embedding and reranking model are missing.
- **Qwen3-Embedding-8B** scores 70.58 on MMTEB (self-reported, Apache-2.0, 4,096 dimensions, 32k tokens) [H1]. The AWS models research put Qwen3-Embedding-0.6B (64.33) on its shortlist because larger Qwen3 embedding models needed a GPU on SageMaker. On Requesty the 8B model costs less than the AWS research's own estimates for self-hosting the 0.6B model. Requesty's catalog lists no context window or batch limit for it, and I did not test it.
- **Speaker turns.** Requesty's transcription endpoint documents `language`, `response_format` and word or segment timestamps, but no speaker labels [R18]. For workshop recordings with speaker turns, Amazon Transcribe stays the only managed option in this comparison.

## 3. Price

**The fee.** The pricing page says that Requesty adds "5% to the base model provider cost", so a model at $10 per million tokens costs $10.50 [R5]. The billing documentation implements it as "A 5% service fee is added to every top-up, so a $100 top-up costs $105 and credits $100 to your balance" [R4]. The catalog prices themselves match the upstream list prices where I could compare them: Claude Haiku 5.5 at $0.11 / $0.55, Claude Sonnet 5.5 at $2.20 / $11.00 and MiniMax M2.5 at $0.36 / $1.44 in eu-central-1 are the same figures as in the AWS models research [R11]. Requesty's EU Claude prices are 10 percent above its global Claude prices ($0.10 against $0.11 for Haiku 5.5), the same regional premium AWS charges.

**Bring-your-own-key.** Requesty supports keys for OpenAI (including a regional endpoint such as `https://eu.api.openai.com/v1`), Anthropic, Google AI Studio, Vertex AI, Mistral and xAI [R6]. AWS Bedrock is not on the list. For traffic on a customer's own key, Requesty's cost chart splits "Requesty Cost" from "Provider Cost", the amount paid "directly to providers through your own keys" [R14]. No page I read states whether Requesty charges a fee on that traffic, so I could not confirm whether the 5 percent applies to it. The question matters only for a route such as an OpenAI project with EU data residency, which ADR-0014 names.

**Monthly cost per task.** The assumptions are the same as in the AWS models research: 500 questions a month for a small and 5,000 for a mid-size project memory, 20,000 input and 1,000 output tokens per answer, 8,000 and 500 per check, 40,000 and 2,000 per planning run, 1,200 judge calls of 6,000 and 500 per bake-off, 30 and 200 million tokens for a first embedding run, and 5 and 40 hours of recordings. The Requesty column includes the 5 percent fee. Token counts for checking, planning and judging are estimates from the AWS models research, so these lines are estimates too.

| Task | Requesty EU model | Requesty, small / mid-size | Bedrock Frankfurt model | Bedrock, small / mid-size |
| --- | --- | --- | --- | --- |
| Answering | Claude Haiku 5.5 | $1.44 / $14.44 | Claude Haiku 5.5 | $1.38 / $13.75 |
| Answering | Qwen3 30B-A3B 2507 (Nebius) | $1.21 / $12.08 | Gemma 4 31B | $1.92 / $19.20 |
| Answering | gpt-oss-120b (Nebius) | $1.89 / $18.90 | gpt-oss-120b | $2.40 / $23.95 |
| Answering | GPT-5 nano (Azure) | $0.81 / $8.09 | Nova Micro | $0.55 / $5.52 |
| Checking (estimate) | Qwen3 235B 2507 (Nebius) | $1.00 / $9.98 | Qwen3 235B 2507 | $1.45 / $14.50 |
| Checking (estimate) | Claude Haiku 5.5 | $0.61 / $6.06 | Claude Haiku 5.5 | $0.58 / $5.78 |
| Judge, per bake-off (estimate) | Claude Sonnet 5.5 | $23.56 | Claude Sonnet 5.5 | $22.44 |
| Planning (estimate) | Claude Haiku 5.5 | $2.89 / $28.88 | Claude Haiku 5.5 | $2.75 / $27.50 |
| Planning (estimate) | Qwen3 235B 2507 (Nebius) | $4.83 / $48.30 | Qwen3 235B 2507 | $6.96 / $69.60 |
| Embedding, first run | Qwen3-Embedding-8B (Nebius) | $0.32 / $2.10 | Cohere Embed v4 | $3.60 / $24.00 |
| Embedding, first run | text-embedding-3-large (Azure France) | $4.10 / $27.30 | Titan Text Embeddings V2 | $6.00 / $40.00 |
| Reranking | none | not available | Amazon Rerank 1.0 | $1.50 / $15.00 |
| Speech to text, one-off | Voxtral Mini Transcribe 3.0, no speaker labels | $1.10 / $8.82 | Amazon Transcribe, 30 speakers | $1.80 / $14.40 |

For the same model, Requesty costs 5 percent more than Bedrock. For the open-weight models that both serve, the EU providers behind Requesty list lower prices than Bedrock: Qwen3 235B 2507 costs $0.20 / $0.60 on Nebius against $0.29 / $1.16 on Bedrock, so the difference outweighs the fee [R11]. Every figure stays small next to the $35.76 a month for the EC2 instance in the AWS installation research. Requesty's catalog also states Claude Haiku 5.5's long-context price, five times the base rate, from 100,000 prompt tokens [R11]. The AWS models research could not find that threshold in the AWS Price List. It is Requesty's figure and not confirmed by AWS.

## 4. What the AWS retrieval services force

**Which calls exist.** A Managed Knowledge Base answers two query calls [A4, A7, A8]:

- `Retrieve` returns chunks with content, metadata, location and score. Its only model parameter for a managed knowledge base is the reranker in `managedSearchConfiguration`: `rerankingModelType` `MANAGED`, `CUSTOM` or `NONE`, and for `CUSTOM` a `modelArn` that must match `arn:aws:bedrock:<region>::foundation-model/...` [A5]. It takes no generation model. It is the retrieve-only call.
- `AgenticRetrieveStream` plans sub-queries, retrieves, judges whether the results suffice, and by default writes an answer with citations. Its `agenticRetrieveConfiguration` takes `foundationModelType` `MANAGED` ("uses the service default") or `CUSTOM`. `CUSTOM` takes a `FoundationModelConfiguration` of type `BEDROCK_FOUNDATION_MODEL` or `MANTLE_FOUNDATION_MODEL` with a model ARN [A6]. It also takes a reranker of type `BEDROCK_RERANKING_MODEL` or the managed one. `generateResponse: false` turns the answer off and returns only the results [A6, A7].
- `RetrieveAndGenerate` "cannot be used with managed knowledge bases" [A8].

The prerequisites for agentic retrieval name "Access to a foundation model in Amazon Bedrock to use for query planning and evaluation" [A7]. AWS adds that "The customer provides and owns the foundation model, embedding model, and reranking model used during agentic retrieval if provided. Your IAM credentials are used to invoke these models" [A7]. A custom planner is billed at "model provider pricing" plus $1 per 1,000 underlying Retrieve calls, and the managed planner at $4 per 1,000 agentic calls plus the same $1 [A9].

**Per step.**

| Step | Service and API field | Model choice | Verdict |
| --- | --- | --- | --- |
| Parsing, Managed KB | Data source `parsingConfiguration`: `SMART_PARSING` (default) or `MULTI_MODAL_EMBEDDINGS`. "Other parsing strategies, such as BEDROCK_FOUNDATION_MODEL and BEDROCK_DATA_AUTOMATION, are not supported" [A3] | Service-managed, unnamed | Fixed by AWS |
| Chunking, Managed KB | `DEFAULT` (300 tokens, 20 percent overlap), `FIXED_SIZE` or `NONE` [A3] | No model | Not a model step. ADR-0017 uses `NONE` |
| Embedding, Managed KB | `CreateKnowledgeBase` `embeddingModelType`: `MANAGED` or `CUSTOM` with Titan Text Embeddings V2, Cohere Embed English v3, Cohere Embed Multilingual v3, Cohere Embed v4 or Nova Multimodal Embeddings at 1,024 dimensions float32, or TwelveLabs Marengo Embed 3.0. "You cannot change the embedding model type after creating the knowledge base" [A2] | Managed or five Bedrock models | Fixed by AWS (default) or chosen from Bedrock only, once |
| Reranking, Managed KB | `rerankingModelType` in `Retrieve` and `AgenticRetrieveStream`: `MANAGED`, `CUSTOM` (Bedrock reranker ARN) or `NONE`. "Managed reranking is available only for knowledge bases that use managed embedding" [A4, A5, A6] | Managed, Bedrock rerankers, or none | Fixed by AWS, chosen from Bedrock only, or free if set to `NONE` on `Retrieve` and the app reranks the returned chunks itself |
| Agentic planning and sufficiency check | `AgenticRetrieveStream` `foundationModelType`: `MANAGED` or `CUSTOM` with a Bedrock or `bedrock-mantle` model ARN [A6, A7] | Managed or Bedrock | Fixed by AWS or chosen from Bedrock only. Free only if the app plans itself and calls `Retrieve`, which no longer tests AWS's agentic retrieval |
| Answer generation | `AgenticRetrieveStream` with `generateResponse: true` uses the planner model. With `false`, or with `Retrieve`, no answer is generated [A6, A7] | Planner model, or any model the app chooses | Free, if the app turns generation off |
| Embedding, customer-managed KB | "Chose any Bedrock embedding model" [A1] | Bedrock | Chosen from Bedrock only |
| Parsing, customer-managed KB | Default parser (no model), a Bedrock foundation model, or Bedrock Data Automation [A1, A12] | Bedrock or AWS service | Chosen from Bedrock only, or fixed by AWS |
| Answer, customer-managed KB | `RetrieveAndGenerate` `modelArn` is a Bedrock model or inference profile [A8]. `Retrieve` returns chunks only | Bedrock, or any model after `Retrieve` | Free with `Retrieve` |
| GraphRAG graph construction | "You choose a foundation model for graph construction when you create a knowledge base" [A11] | Bedrock | Chosen from Bedrock only |
| Standalone reranking | Bedrock `Rerank` API: Amazon Rerank 1.0, Cohere Rerank 3.5 (AWS models research) | Bedrock | Chosen from Bedrock only. Requesty has no substitute |

**Can the knowledge base be used for retrieval only?** Yes. `Retrieve` returns chunks and takes no generation model, and `AgenticRetrieveStream` with `generateResponse: false` returns the deduplicated results of all its iterations and no answer [A4, A7]. The answer model can then be any LiteLLM route, a Requesty route included. The planner inside `AgenticRetrieveStream` cannot be a Requesty route.

**One correction to ADR-0017.** ADR-0017 says the knowledge base embeds each record "with its own model, which cannot be changed". The AWS documentation read on 2026-11-21 offers a custom Bedrock embedding model at creation, at the cost of the managed reranker [A2, A4]. The model still cannot be changed after creation. This contradicts the wording of ADR-0017, not its decision.

Two details I could not confirm:

- Whether the `CUSTOM` planner accepts an EU geo inference profile, such as Claude Haiku 5.5 in Frankfurt. The ARN pattern allows a resource prefix such as `inference-profile/` [A6], but the documentation shows no example. Gemma 4 on `bedrock-mantle` runs in-region in Frankfurt and fits the `MANTLE_FOUNDATION_MODEL` type.
- Which models AWS uses for `MANAGED` parsing, embedding, reranking and planning, and in which region they run. AWS names none of them.

## 5. Lock-in

If every model call the app controls goes over Requesty, these stay on AWS:

- The Managed Knowledge Base itself: raw-data storage at $5 per GB a month, the managed parser, the managed or Bedrock embedding, the reranker, and the planner of `AgenticRetrieveStream` [A9].
- Reranking for the self-built arms, because Requesty serves no reranker. The alternative is a self-hosted reranker, which the AWS models research priced at $672 a month or more on a GPU.
- Speech to text with speaker labels, through Amazon Transcribe.
- The EC2 instance, S3 and the chunk store (AWS installation research).

What leaving AWS would still cost:

- The knowledge base's vectors cannot be exported (cloud provider research), so the managed arm ends with AWS. ADR-0017 already treats them as a cache.
- The chunk text in the app's own store is the durable layer (ADR-0013), and it does not change with Requesty.
- The self-built arms' vectors move with them if their embedding model is open-weight and runs elsewhere. Qwen3-Embedding-8B over Requesty fits, because the same weights run on-device [H1]. Cohere Embed v4 does not, because it exists only on Bedrock among these options. A move after Cohere v4 costs one re-embedding run, which ADR-0025 covers.
- The Requesty routes themselves move with LiteLLM unchanged, because they do not depend on AWS. Requesty adds a small lock-in of its own: model IDs such as `nebius/qwen/qwen3-235b-a22b-instruct-2507` and the prepaid balance. The API is OpenAI-compatible, so another gateway or a direct provider route replaces it by changing `api_base` and the model name (inferred from [R1, L2]).

In short, Requesty does not reduce the AWS lock-in of the managed arm. It removes Bedrock as the only source of models for everything else, which ADR-0013 already keeps free of AWS-only parts.

## 6. LiteLLM

**How LiteLLM reaches Requesty.** LiteLLM has no native Requesty provider. Its `litellm/llms` directory and its JSON list of 43 OpenAI-like providers contain no Requesty entry, and a code search of the repository finds no match, at commit `6dea2b5` of 2026-11-21 (release v1.110.2) [L1]. The route is LiteLLM's generic OpenAI-compatible provider, `openai/<model>` with `api_base` ending in `/v1` [L2]. For example (not tested):

```yaml
model_list:
  - model_name: check
    litellm_params:
      model: openai/nebius/qwen/qwen3-235b-a22b-instruct-2507
      api_base: https://router.eu.requesty.ai/v1
      api_key: os.environ/REQUESTY_API_KEY
    model_info:
      model_location: third-party-cloud
      input_cost_per_token: 0.00000021    # $0.20 per million plus 5 percent
      output_cost_per_token: 0.00000063   # $0.60 per million plus 5 percent
```

The prices must be set by hand. LiteLLM's price table does not know Requesty's model IDs, and LiteLLM's custom pricing page notes a case where a cost lookup that finds no row "bills the request at $0" [L3]. A route billed at $0 would escape the LiteLLM budget, which is the installation's immediate spend stop (AWS installation research). This is my inference for Requesty routes in general.

**What Requesty adds or duplicates.**

| Feature | Requesty | LiteLLM | Effect of two gateways |
| --- | --- | --- | --- |
| Fallbacks | Fallback, load-balancing and latency policies, and managed `@eu` policies that "route only to EU-hosted endpoints and fail over between them" [R1, R15] | Router fallbacks and retries | Retries multiply if both retry. Keep fallbacks in one layer |
| Budgets | Monthly cap per API key, and the prepaid balance, which stops calls with HTTP 402 [R4, R14] | Global, key, team and user budgets in PostgreSQL | Requesty's balance is a second hard stop. LiteLLM's budget only works with prices set |
| Caching | Provider prompt caching, with breakpoints added on request through `requesty.auto_cache` [R16] | Response cache, and prompt caching passed through | Different layers. No conflict found |
| Logging | Telemetry always, payloads for 30 days unless switched off [R3] | Spend logs in its database | A second copy of telemetry outside the installation |
| Model policy | Approved models, hosting location and serving region per organisation [R2] | `model_info.model_location` per route, read by the app | Requesty's rules are a second guard for third-party-cloud routes only |

**Streaming, tool calls, structured output, rerank, embedding.** Requesty documents streaming over server-sent events, usage in the stream only with `stream_options: {"include_usage": true}`, strict `json_schema` output, and tool calling per model [R11, R17]. For Claude models it removes JSON Schema keywords that Anthropic does not support and moves their values into the property description [R17]. Its OpenAPI specification has `/v1/chat/completions`, `/v1/messages`, `/v1/responses`, `/v1/embeddings` and `/v1/audio/transcriptions`, but no rerank path [R12]. So:

- Chat, tool calls and structured output should pass through two OpenAI-compatible gateways in a row. Nobody has tested it for this installation, and the schema rewriting for Claude means that a strict schema is not fully enforced on Claude routes.
- Streaming should work. Whether LiteLLM's `openai/` provider sends `stream_options` so that LiteLLM sees token usage for its budget is not confirmed.
- Embedding works over `/v1/embeddings`.
- Rerank cannot go over Requesty. LiteLLM must keep a direct Bedrock rerank route.
- LiteLLM's vector store integration for Bedrock knowledge bases builds a `vectorSearchConfiguration` when it sets the result count or filters [L4]. AWS says that field "applies only to custom knowledge bases" [A2], and LiteLLM's code has no `managedSearchConfiguration` or `AgenticRetrieveStream` [L1, L4]. The app should call the knowledge base with the AWS SDK, not through LiteLLM (inferred).

**Could Requesty replace LiteLLM?** No:

- Requesty has no per-route model location that the app can read. Its catalog field `geolocation` takes the values `eu`, `us`, `global` and others [R11], but the glossary's model locations are on-device, client-hosted and third-party cloud. Every Requesty route is third-party cloud.
- Requesty cannot reach on-device or client-hosted models. "Custom model hosting" appears only in the Enterprise column of the pricing page [R5].
- Requesty cannot call Bedrock in our own account, so Bedrock rerank, Gemma 4 on `bedrock-mantle`, Amazon Transcribe and the knowledge base still need direct AWS routes [R6, R11].
- The app's check for off-policy calls reads LiteLLM's `/model/info` (model server research). With Requesty alone, that check would move into Requesty's configuration, outside the installation.

## 7. The bake-off

ADR-0016 gives every arm the same models. If the self-built arms call models over Requesty and the managed arm's own steps call Bedrock, these differences come from models, not from retrieval:

- **Embedding.** The managed arm embeds with the managed model or a Bedrock model. The self-built arms embed with whatever their route serves, for example Qwen3-Embedding-8B over Requesty. The two can only be equal on Bedrock (Cohere Embed v4 or Titan V2), and a custom embedding model costs the managed arm its managed reranker [A2, A4].
- **Reranking.** The managed reranker is unnamed. The same reranker for all arms means a Bedrock reranker for all arms: `CUSTOM` in the managed arm and the Bedrock `Rerank` API in the self-built arms. With `Retrieve` and `rerankingModelType: NONE`, the app could rerank the managed arm's chunks with the shared reranker itself.
- **Planning.** The managed planner is unnamed. The same planner means a model that Bedrock serves, set as `CUSTOM`, and the self-built agent arms must then use that model too. Candidates on both sides are Claude Haiku 5.5 and Gemma 4 on `bedrock-mantle`; Requesty serves only Haiku 5.5 of these. Even with the same model, AWS writes the planner's prompts, and those count as part of the managed retrieval.
- **Answer.** With `generateResponse: true`, the planner model writes the managed arm's answer with AWS's prompt and citation format. With `generateResponse: false`, the app writes every arm's answer with the same answer model and prompt.
- **The same model over two accounts.** Claude Haiku 5.5 over Requesty runs in Requesty's AWS account, and the managed arm's Haiku 5.5 runs in ours. The weights are the same, but sampling settings, reasoning effort and the EU geo routing may differ. The bake-off should fix those settings in both places.
- **Off-policy gate.** Calls inside the knowledge base never pass LiteLLM, so the gate of 0 off-policy model calls cannot see them. They run in Frankfurt in the consultancy's tenant, which ADR-0013 allows, but the app has to record them as third-party cloud from configuration, not from `/model/info`.

## Limits and open points

- I did not create a Requesty account or key. Everything about Requesty comes from its public documentation, legal pages and its public model catalog. An organisation's approved-model list, the DPA PDF and the Compliance page need a login, and I did not read them.
- I could not read trust.requesty.ai, so the SOC 2 status rests on Requesty's security page.
- Requesty's EU guarantees are its own statements. No audit report confirms them.
- I could not confirm whether the 5 percent fee applies to bring-your-own-key traffic.
- I could not confirm whether Requesty's `@eu-central-1` Claude routes use Bedrock in-region or EU geo inference.
- I could not confirm whether the knowledge base's `CUSTOM` planner accepts an EU geo inference profile, or which models and regions the `MANAGED` steps use.
- Nothing was tested: not two gateways in a row, not streaming usage, not tool calls through both, not Qwen3-Embedding-8B's limits, not latency. Requesty's Frankfurt gateway and an EC2 instance in Frankfurt are in the same AWS region, so the extra hop should be short, but nobody has measured it.
- Requesty publishes no uptime figure for pay-as-you-go plans.
- The catalog changes often. The counts and prices here are a snapshot of 2026-11-21.
- Requesty's `vertex/google/gemini-embedding-001@europe-*` routes carry `geolocation: global`, which looks like a labelling error.

## Recommendations

These are research findings, not owner decisions.

1. Keep LiteLLM as the only model API of the app, and add Requesty only as upstream routes behind it, with `model_info.model_location: third-party-cloud` and hand-set prices that include the 5 percent fee.
2. If Requesty is used, set the serving region to EU, the model hosting location to EU, EU-only approved models and zero data retention before the first request, and sign the DPA.
3. Call the Managed Knowledge Base directly with the AWS SDK, use `AgenticRetrieveStream` with `generateResponse: false`, and let the shared answer model write every arm's answer.
4. To keep ADR-0016's "same models" as close as the knowledge base allows, set the managed arm's planner to `CUSTOM` with a Bedrock model that the self-built agent arms also use, and use a Bedrock reranker in all arms. Accept the embedding as part of the managed retrieval, as ADR-0017 does, or test a custom Cohere Embed v4 knowledge base as a second managed arm.
5. Consider Qwen3-Embedding-8B over Requesty and Qwen3 235B 2507 on Nebius as additional bake-off candidates for the self-built arms. Both cost less than their Bedrock counterparts, and the embedding model keeps vectors portable.
6. Keep Gemma 4, rerank and speech to text with speaker labels on Bedrock and Transcribe, because Requesty EU has no equivalent.
7. Raise the ADR-0017 wording on the embedding model in the review of [Review and adopt: models per task on AWS](https://github.com/Niko4417/OPT-cc/issues/43).

## Method

I read Requesty's documentation index and pages, its pricing, DPA, privacy, security and sub-processor pages, and its inference OpenAPI specification on 2026-11-21. I downloaded the public model catalog from `https://router.eu.requesty.ai/v1/models/all` without a key; it was byte-identical to the global endpoint's catalog. I counted models by `api` and `geolocation` and read prices from each entry's `pricing`. For AWS I read the Bedrock user guide pages on managed knowledge bases, agentic retrieval, ingestion and GraphRAG, the Agents for Amazon Bedrock Runtime API reference for `Retrieve`, `AgenticRetrieveStream`, `RetrieveAndGenerate` and their data types, and the Bedrock pricing page. For LiteLLM I read its `litellm/llms` directory, its OpenAI-like provider list and its Bedrock vector store code at commit `6dea2b5`, and its documentation on OpenAI-compatible endpoints and custom pricing. Prices are list prices in USD without VAT. Monthly costs reuse the assumptions of the AWS models research, and every token count for checking, planning and judging is an estimate. I created no account and sent no request with a key.

## Sources

All read on 2026-11-21.

- R1: [EU Routing](https://docs.requesty.ai/features/eu-routing), Requesty docs
- R2: [Compliance](https://docs.requesty.ai/features/compliance-report), Requesty docs
- R3: [Data Privacy and Zero Data Retention](https://docs.requesty.ai/features/data-privacy), Requesty docs
- R4: [Billing](https://docs.requesty.ai/features/billing), Requesty docs
- R5: [Pricing](https://www.requesty.ai/pricing), including its FAQ answers
- R6: [Bring Your Own Keys](https://docs.requesty.ai/features/bring-your-own-keys), Requesty docs
- R7: [Data Processing Agreement](https://www.requesty.ai/dpa)
- R8: [Sub-processors](https://www.requesty.ai/privacy/subprocessors), last updated 2026-10-29
- R9: [Privacy Policy](https://www.requesty.ai/privacy)
- R10: [Security](https://www.requesty.ai/security)
- R11: Public model catalog, [`/v1/models/all` on the EU endpoint](https://router.eu.requesty.ai/v1/models/all), and [List Models](https://docs.requesty.ai/api-reference/endpoint/models-list), Requesty docs
- R12: [Inference API OpenAPI specification](https://docs.requesty.ai/api-reference/requesty_inference-openapi.json) and [Inference APIs overview](https://docs.requesty.ai/api-reference/inference-apis)
- R13: [In-Flight Rate Limits](https://docs.requesty.ai/features/in-flight-rate-limits), Requesty docs
- R14: [Spend Limits and Rate Limits](https://docs.requesty.ai/features/api-limits) and [Cost Tracking](https://docs.requesty.ai/features/cost-tracking), Requesty docs
- R15: [Fallback Policies](https://docs.requesty.ai/features/fallback-policies) and [Managed Policies](https://docs.requesty.ai/features/managed-policies), Requesty docs
- R16: [Auto Caching](https://docs.requesty.ai/features/auto-caching), Requesty docs
- R17: [Streaming](https://docs.requesty.ai/features/streaming) and [Structured Outputs](https://docs.requesty.ai/features/structured-outputs), Requesty docs
- R18: [Create Transcription](https://docs.requesty.ai/api-reference/endpoint/audio-transcriptions-create), Requesty docs
- A1: [Build a managed knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-build-managed.html), Amazon Bedrock user guide
- A2: [Create a managed knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-create.html)
- A3: [Customize ingestion for a data source (managed)](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-customize-ingestion.html)
- A4: [Query a knowledge base and retrieve data](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve.html)
- A5: [Retrieve](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_Retrieve.html), [ManagedSearchConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_ManagedSearchConfiguration.html) and [ManagedSearchBedrockRerankingModelConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_ManagedSearchBedrockRerankingModelConfiguration.html), API reference
- A6: [AgenticRetrieveStream](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_AgenticRetrieveStream.html), [AgenticRetrieveConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_AgenticRetrieveConfiguration.html), [FoundationModelConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_FoundationModelConfiguration.html), [BedrockFoundationModelModelConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_BedrockFoundationModelModelConfiguration.html) and [AgenticRetrieveRerankingConfiguration](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_AgenticRetrieveRerankingConfiguration.html), API reference
- A7: [Use agentic retrieval to query a knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-agentic-retrieve.html)
- A8: [RetrieveAndGenerate](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html), API reference
- A9: [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), Managed Knowledge Base section
- A10: [Supported AWS Regions for managed knowledge bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-regions.html) and [Service quotas for managed knowledge bases](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-quotas.html)
- A11: [Build a knowledge base with Amazon Neptune Analytics graphs](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-build-graphs.html)
- A12: [Parsing options for your data source](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-advanced-parsing.html)
- L1: LiteLLM repository at commit [`6dea2b5`](https://github.com/BerriAI/litellm/tree/6dea2b5a72b678db85ee50a59d1b23df23dd5eef): [`litellm/llms`](https://github.com/BerriAI/litellm/tree/6dea2b5a72b678db85ee50a59d1b23df23dd5eef/litellm/llms) and [`litellm/llms/openai_like/providers.json`](https://github.com/BerriAI/litellm/blob/6dea2b5a72b678db85ee50a59d1b23df23dd5eef/litellm/llms/openai_like/providers.json), plus a GitHub code search for "requesty" in the repository
- L2: [OpenAI-Compatible Endpoints](https://docs.litellm.ai/docs/providers/openai_compatible), LiteLLM docs
- L3: [Custom pricing](https://docs.litellm.ai/docs/proxy/custom_pricing), LiteLLM docs
- L4: [`litellm/llms/bedrock/vector_stores/transformation.py`](https://github.com/BerriAI/litellm/blob/6dea2b5a72b678db85ee50a59d1b23df23dd5eef/litellm/llms/bedrock/vector_stores/transformation.py) at commit `6dea2b5`
- E1: European Commission, [Daily News 19/12/2025](https://ec.europa.eu/commission/presscorner/detail/en/mex_25_3136), renewal of the UK adequacy decisions until 2031-12-27
- H1: [Qwen/Qwen3-Embedding-8B model card](https://huggingface.co/Qwen/Qwen3-Embedding-8B)
- Project research: [AWS models](https://github.com/Niko4417/OPT-cc/blob/research/aws-models/docs/research/aws-models.md), [cloud provider](https://github.com/Niko4417/OPT-cc/blob/research/cloud-provider/docs/research/cloud-provider.md), [AWS installation](https://github.com/Niko4417/OPT-cc/blob/research/aws-installation/docs/research/aws-installation.md), [model server](https://github.com/Niko4417/OPT-cc/blob/research/model-server/docs/research/model-server.md)
