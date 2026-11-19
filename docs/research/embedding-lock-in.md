# Embedding model independence of a project memory

Date: 2026-11-19. Ticket: [#34](https://github.com/Niko4417/OPT-cc/issues/34), map [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

How can a project memory stay independent of its embedding model? Is the Keiko team's claim true that "a pod must be changed with the embedding model that built it"? What do these options cost and keep: re-embedding from stored chunk text, model-free lexical search (BM25), learned sparse retrieval (SPLADE), translating vectors between models (vec2vec, adapters), and late interaction (ColBERT)? How do Azure AI Search with Foundry IQ, Bedrock Knowledge Bases and OVHcloud AI Endpoints handle an embedding model change? Which option should the bundle format and the bake-off assume?

## Answer

Keep the chunk text as the durable layer, treat vectors as a rebuildable cache, and re-embed on every model change. This is the plan the [bundle format research](https://github.com/Niko4417/OPT-cc/blob/research/bundle-format/docs/research/bundle-format.md) already makes. This note confirms it with numbers, adds two fields to the bundle format and three measurements to the bake-off. No other option removes the dependency on the embedding model, and none is cheaper than re-embedding at the size of a project memory.

Verdict on the Keiko team's claim: partly confirmed. Vectors are bound to the model that produced them. Keiko's own code, Azure, AWS and the vec2vec results all agree on that. The pod itself is not bound to the model:

- Keiko keeps a capsule usable through lexical retrieval when its embedding model is missing or incompatible.
- Keiko's "force re-embed" path rebinds an existing capsule to a new model without creating a new one.
- Re-embedding a mid-size project memory costs minutes and less than 2 euros.

Re-embedding cost for a mid-size project memory (30 MB of text, about 8.3 million tokens) and a large one (300 MB, about 83 million tokens):

| Where | Model | Mid-size cost | Mid-size time | Large cost | Large time |
| --- | --- | ---: | ---: | ---: | ---: |
| On-device, M5 Pro | Qwen3-Embedding-0.6B | electricity only | 7 to 28 min (estimate) | electricity only | 1.1 to 4.6 h (estimate) |
| On-device, M5 Pro | Qwen3-Embedding-4B | electricity only | 0.9 to 3.7 h (estimate) | electricity only | 9 to 37 h (estimate) |
| Azure, Data Zone | text-embedding-3-small | $0.18 | 8 min at 1M tokens per minute | $1.82 | 83 min |
| Azure, Data Zone | text-embedding-3-large | $1.18 | 8 min | $11.80 | 83 min |
| AWS Bedrock, Frankfurt | Titan Text Embeddings V2 | $1.65 ($0.83 batch) | bound by a request quota | $16.50 ($8.25 batch) | bound by a request quota |
| AWS Bedrock, Frankfurt | Cohere Embed v4 | $0.99 | bound by a request quota | $9.90 | bound by a request quota |
| OVHcloud AI Endpoints | bge-m3 | EUR 0.08 | 2 to 50 min at 400 requests per minute | EUR 0.83 | 16 min to 8.3 h |
| OVHcloud AI Endpoints | Qwen3-Embedding-8B | EUR 0.83 | 2 to 50 min | EUR 8.25 | 16 min to 8.3 h |

Prices exclude VAT. The on-device times are estimates from measured prefill rates, not an embedding run. The bake-off must measure them.

How much quality survives without the embedding model, measured on the same MTEB harness (nDCG@10, section 2):

| Task group | BM25 (no model) | Qwen3-Embedding-0.6B (dense) | GTE-ModernColBERT (late interaction) |
| --- | ---: | ---: | ---: |
| 12 code retrieval tasks of MTEB(Code, v1) | 44.4 | 75.4 | not run |
| 14 BEIR prose tasks | 41.1 | 56.8 | 55.4 |
| 5 German retrieval tasks | 65.2 | 79.9 | not run |

Lexical-only search is a usable fallback but not an equal alternative. It loses about 31 points on code tasks that ask in natural language, about 16 points on English prose and about 15 points on German. BM25 wins on one task, German legal text.

The other options do not solve lock-in:

- SPLADE is a learned model. Its sparse vectors are as tied to the model as dense ones. The best code model, SPLADE-Code, has a non-commercial license (CC BY-NC-SA 4.0).
- vec2vec and adapters translate vectors between models. vec2vec reaches a cosine of 0.74 to 0.92 with the true target vectors and was never tested on retrieval quality (nDCG). Adapters recover 95% to 99% of recall for closely related models and 59% to 72% for distant ones. Both need training data in the new model's space: tens of thousands of embeddings, which is as many as the 20,000 chunks of a mid-size project memory. Re-embedding the chunk text instead is exact and needs no more embedding calls.
- Late interaction (ColBERT) stores one vector per token. That is 2 to 26 times more data than one dense vector per chunk, and every vector is tied to the ColBERT model. It increases lock-in.

The managed services do not re-embed for you either:

- Azure says "Changing a vectorizer or deployment name doesn't regenerate indexed vectors". A change of embedding model needs a new index or a new vector field, filled again from source.
- Bedrock cannot change the embedding configuration of an existing knowledge base. A new model means a new knowledge base and a full ingestion.
- OVHcloud AI Endpoints sells embeddings only and leaves re-embedding to the app.

What the bundle format and the bake-off should assume:

1. Hybrid retrieval with a BM25 lane and one dense lane. Chunk text is the source of truth. Vectors are reused only on an exact match and otherwise rebuilt, as the bundle research already specifies.
2. Two additions to the bundle format. First, record the lexical analyzer (tokenizer, stemmer, stop words, BM25 parameters) in `memory.json`, so that the lexical-only fallback gives the same results on every machine. Second, add a behavioral probe to the embedding-space fingerprint, as Keiko does, so a provider that changes its model behind a stable name is caught.
3. Do not add vector translation, adapters, SPLADE or ColBERT to the bundle format. ColBERT and SPLADE may still enter the bake-off as quality arms. Rerankers read text at query time, store nothing and can change freely.
4. The bake-off measures three things the literature cannot settle for this project: re-embedding throughput on the target machine and at the chosen provider; the lexical-only arm on the gold question set; and a model-swap run that re-embeds with a second model and compares answers.

## 1. What the Keiko team's claim rests on

Keiko `dev` at commit [`bc57815`](https://github.com/oscharko-dev/Keiko/tree/bc57815cbb383faa6531dd416afa5fe6ccd3d56a) (2026-11-19), read through `gh`.

**What confirms it.** Keiko pins a capsule to one embedding model. Issue [#4338](https://github.com/oscharko-dev/Keiko/issues/4338) (open, verified on `dev` 2026-11-10) says the first verified model "wins, and the pod is pinned to it for good: provider, model id, dimensions and metric". Dense lanes whose identity does not match "fail closed for vector retrieval" ([`knowledge-pods.md`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/local-knowledge/knowledge-pods.md), "Retrieval and score semantics"). Raw vector scores "remain comparable only inside one embedding space". So vectors cannot outlive their model, and the external evidence below agrees.

**What contradicts it.** The same documents show the capsule outliving its model:

- A capsule with an unknown or incompatible embedding profile stays "usable through lexical retrieval", and the UI shows reindex guidance (`knowledge-pods.md`). The embedding governance ledger lists "Lexical candidates survive when dense embedding cannot run" as a tested scenario ([`embedding-space-governance-ledger.md`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/local-knowledge/embedding-space-governance-ledger.md)).
- The reindex route has a `full-reembed` mode. `rebindCapsuleEmbeddingIdentityForForceReembed` verifies the new model and calls `updateCapsuleEmbeddingModelIdentity` on the existing capsule ([`local-knowledge-handlers.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-server/src/local-knowledge-handlers.ts), lines 478 to 500). Issue #4338 plans to keep this as the explicit way to switch an existing pod's model.
- The indexing orchestrator has a recovery mode that "re-embeds using existing chunks only" ([`orchestrator.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-local-knowledge/src/indexing/orchestrator.ts), header comment). Its force mode, which a model switch uses, re-chunks "from the current source text". Text-like documents "are re-read from disk". So in Keiko a model switch needs the original files to be reachable. The bundle design avoids that need because it ships the chunk text.
- Keiko does not reindex private content on its own ("This change does not automatically reindex private content", governance ledger, "Known limits"). That is a policy choice about consent, not a technical lock.

**Keiko's fingerprint.** Keiko's `embeddingSpaceFingerprintForVectors` embeds four fixed probe strings, keeps 16 sampled lanes of each normalized vector plus the pairwise cosines, rounds them to four decimals and hashes the result ([`embedding.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-model-gateway/src/embedding.ts), lines 197 to 239). This is a behavioral fingerprint. It detects a changed model even when the provider keeps the model name. The bundle research describes a SHA-256 over metadata fields only. A metadata hash cannot see a silent change behind a stable name, such as Bedrock's service-managed embedding model (section 5). Rounding to four decimals may also differ between Metal and CUDA builds of the same model, which would force a needless re-embed. Comparing probe vectors by cosine (for example at least 0.999) avoids that. That threshold is my proposal and is untested.

## 2. What survives without an embedding model

All numbers in this section come from one harness: the MTEB results repository at commit [`cde9311`](https://github.com/embeddings-benchmark/results/tree/cde931100092c418d944693da9193b012b15af38/results) (2026-11-19). The metric is nDCG@10 times 100 on the test split, averaged over a task's subsets where it has several. BM25 is the `mteb/baseline-bm25s` run (the [bm25s](https://github.com/xhluca/bm25s) library, revisions `0_1_10` and `0_3_0`). Qwen3-Embedding-0.6B is the dense model the map already chose (ticket #4).

### Code

| Task | BM25 | SPLADE-Code 0.6B | SPLADE-Code 8B | Qwen3-Embedding-0.6B |
| --- | ---: | ---: | ---: | ---: |
| AppsRetrieval | 4.8 | 66.4 | 87.2 | 75.3 |
| COIRCodeSearchNetRetrieval | 40.9 | 86.2 | 89.2 | 84.7 |
| CodeEditSearchRetrieval | 49.9 | 68.4 | 76.5 | 64.4 |
| CodeFeedbackMT | 59.2 | 91.6 | 93.4 | 90.8 |
| CodeFeedbackST | 68.2 | 84.7 | 87.3 | 86.4 |
| CodeSearchNetCCRetrieval | 54.0 | 91.4 | 94.8 | 91.7 |
| CodeSearchNetRetrieval | 60.0 | 90.5 | 92.0 | 91.0 |
| CodeTransOceanContest | 47.8 | 90.9 | 94.4 | 86.1 |
| CodeTransOceanDL | 34.4 | 31.7 | 26.6 | 31.4 |
| CosQA | 18.8 | 37.1 | 33.6 | 36.5 |
| StackOverflowQA | 70.3 | 93.4 | 96.6 | 90.0 |
| SyntheticText2SQL | 24.9 | 74.6 | 75.3 | 76.7 |
| Mean of 12 | 44.4 | 75.6 | 78.9 | 75.4 |

The CoIR paper ([arXiv 2407.02883](https://arxiv.org/abs/2407.02883), ACL 2025, v3 2025-06-06) reports the same pattern on its ten tasks: BM25 averages 29.8 against 45.6 for OpenAI ada-002 and 56.3 for Voyage-Code-002. The Qwen3 Embedding report ([arXiv 2506.05176](https://arxiv.org/abs/2506.05176), v3 2025-06-11, Table 9) gives the same 75.41 for the 0.6B model.

These tasks mostly ask for code in natural language ("find a function that ...") or match code to code. They do not test the lookups where lexical search is strong, such as an identifier, an error message or a ticket key. The code retrieval ticket covers those. On the tasks measured here, a project memory without its embedding model finds code much worse.

### English prose (BEIR)

| Task | BM25 | GTE-ModernColBERT | Qwen3-Embedding-0.6B |
| --- | ---: | ---: | ---: |
| ArguAna | 49.3 | 47.5 | 71.0 |
| ClimateFEVER | 13.6 | 31.3 | 42.1 |
| CQADupstackRetrieval | 31.9 | 41.1 | 46.0 |
| DBPedia | 29.9 | 47.6 | 39.5 |
| FEVER | 48.1 | 87.7 | 88.2 |
| FiQA2018 | 25.1 | 45.3 | 46.6 |
| HotpotQA | 56.9 | 77.5 | 65.7 |
| NFCorpus | 32.1 | 37.8 | 36.7 |
| NQ | 28.5 | 61.6 | 53.5 |
| QuoraRetrieval | 80.4 | 86.7 | 87.8 |
| SCIDOCS | 15.8 | 19.2 | 24.4 |
| SciFact | 68.7 | 76.3 | 69.7 |
| TRECCOVID | 62.3 | 84.8 | 90.5 |
| Touche2020 | 33.1 | 31.3 | 33.2 |
| Mean of 14 | 41.1 | 55.4 | 56.8 |

BM25 here agrees with the BEIR paper ([arXiv 2104.08663](https://arxiv.org/abs/2104.08663), NeurIPS 2021 Datasets and Benchmarks), whose 18 datasets give BM25 a mean of 42.3 (my mean of the paper's table). BEIR called BM25 "a robust baseline" in 2021, when dense models of that time did not beat it on average (TAS-B at 41.5). Current dense models beat it by 15 points.

### German

| Task | BM25 | Qwen3-Embedding-0.6B |
| --- | ---: | ---: |
| GermanQuAD-Retrieval | 90.0 | 92.0 |
| GermanDPR | 45.5 | 79.1 |
| German1Retrieval | 86.5 | 95.3 |
| GermanHealthcare1Retrieval | 37.3 | 71.4 |
| GermanLegal1Retrieval | 66.9 | 61.8 |
| Mean of 5 | 65.2 | 79.9 |

The bm25s baseline in MTEB is configured for English (`languages: eng-Latn` in its `model_meta.json`). A German stemmer may narrow the gap. I did not test that.

### Learned sparse retrieval

SPLADE produces one weight per vocabulary term, not a dense vector, and a standard inverted index can serve it. It is still a neural model. SPLADE-Code 0.6B has 596 million parameters and a vocabulary of 151,936 terms (MTEB `model_meta.json`). Its term weights come from that model, and another SPLADE model produces different weights. Changing it means re-encoding everything, as with a dense model. On code, SPLADE-Code 0.6B (75.6) ties Qwen3-Embedding-0.6B (75.4), and the 8B model reaches 78.9 ([arXiv 2603.22008](https://arxiv.org/abs/2603.22008), 2026-03-23). Both SPLADE-Code models are licensed CC BY-NC-SA 4.0 ([model card](https://huggingface.co/naver/splade-code-06B)), so a consultancy cannot use them commercially. On English prose, SPLADE-v3 reaches a BEIR-13 mean of 51.7 ([arXiv 2403.06789](https://arxiv.org/abs/2403.06789), 2024-03-11). BGE-M3 produces dense, sparse and multi-vector output from one model ([model card](https://huggingface.co/BAAI/bge-m3)), and all three are tied to it.

Learned sparse retrieval changes where vectors live, not who produced them. It is not a way out of lock-in.

## 3. Translating vectors between models

### vec2vec

vec2vec ([arXiv 2505.12540](https://arxiv.org/abs/2505.12540), Jha, Zhang, Shmatikov and Morris, v1 2025-05-18, v4 2026-01-26) learns a translation between two embedding spaces without paired data, using adversarial and cycle-consistency losses. Results from v4:

- On Natural Questions, in distribution, the mean cosine between translated and true target vectors ranges from 0.74 (e5 to gtr) to 0.92 (stella to gte). Top-1 accuracy is 0.82 to 1.00. That means picking the right counterpart among 8,192 candidates, which measures matching, not retrieval.
- Out of distribution, on medical records (MIMIC), the worst pairs fall to a top-1 accuracy of 0.01 and a mean rank of 1,030 to 1,414 out of 8,192.
- Translating into Qwen gives a cosine of only 0.50 (gte to qwen), although top-1 accuracy is 0.92.
- The paper reports no nDCG, BEIR or MTEB retrieval result. Its only retrieval test is text to image on MS COCO, where Recall@16 is 0.23 against 0.75 for the native model.
- Training needs 1 to 7 GPU-days per pair. With 10,000 training embeddings it fails (cosine 0.57, top-1 0.01). With 50,000 it reaches 0.74 and 0.81. For cross-backbone pairs, "only 3 out of 15 runs" converged.

mini-vec2vec ([arXiv 2510.02348](https://arxiv.org/abs/2510.02348), Dar, v4 2026-02-17) replaces the adversarial training with a linear map. It runs in "less than ten minutes" on a CPU from about 60,000 unpaired sentences, and matches or beats vec2vec on top-1 accuracy and rank. It too reports no nDCG or recall, and evaluates on one dataset.

### Adapters with paired data

Drift-Adapter ([arXiv 2509.23471](https://arxiv.org/abs/2509.23471), Vejendla, EMNLP 2025) trains a small map from paired old and new embeddings, so new queries can search old vectors. On 1 million items with 20,000 pairs, it recovers 95.3% to 99.2% of the Recall@10 of full re-embedding between all-MiniLM-L6-v2 and all-mpnet-base-v2. For a distant jump (GloVe to MPNet) it recovers only 58.7% to 71.5%. The paper says the method "cannot be trained" when the old model is unavailable, and that full re-embedding is still needed "for peak performance".

### Fit for a project memory

Both methods pay off when re-embedding a corpus costs a lot and the corpus is much larger than the training set. A project memory is the opposite case. The mid-size one has 20,000 chunks. Drift-Adapter needs 20,000 pairs and mini-vec2vec about 60,000 unpaired sentences split across both spaces, so the training set is as large as the corpus or larger. A project memory always keeps its chunk text, so exact re-embedding is always possible. It costs at most a few euros (see the Answer table). Translation would trade a cheap exact rebuild for a lossy approximation that nobody has measured on retrieval.

The vec2vec paper also shows a risk. An attacker with only the vectors could translate them and infer content: "as many as 80% of emails" in one setting. Vectors in a bundle are therefore as sensitive as the text, which the bundle research already assumes by encrypting the whole bundle.

## 4. Late interaction

ColBERT stores one small vector per token and scores with MaxSim. ColBERTv2 ([arXiv 2112.01488](https://arxiv.org/abs/2112.01488), NAACL 2022) compresses each 128-dimension token vector to 20 bytes (1-bit residual) or 36 bytes (2-bit), down from 256 bytes at 16-bit. On MS MARCO the index shrinks from 154 GiB to 16 to 25 GiB.

For the project memories here:

| Representation | Mid-size (8.3M tokens, 20,000 chunks) | Large (83M tokens, 200,000 chunks) |
| --- | ---: | ---: |
| One dense vector per chunk, 1,024 float32 | 82 MB | 819 MB |
| ColBERTv2, 1-bit residual | 165 MB | 1.65 GB |
| ColBERTv2, 2-bit residual | 297 MB | 2.97 GB |
| Token vectors, 128 float16, uncompressed | 2.1 GB | 21 GB |

Every token vector comes from the ColBERT model and its tokenizer. A model change re-encodes every token. The compressed form also depends on centroids trained on the corpus, so it is tied to one index build as well. Late interaction therefore makes the vector part of a bundle 2 to 26 times larger and keeps the same lock-in.

On quality it does not beat a modern dense model. GTE-ModernColBERT (149M parameters, Apache-2.0) reaches 55.4 on the 14 BEIR tasks against 56.8 for Qwen3-Embedding-0.6B (section 2). It wins on five tasks, among them DBPedia, HotpotQA and NQ, and loses on the other nine. No late-interaction code model has results in the MTEB repository.

A ColBERT model or a cross-encoder can also rerank at query time from text alone, without storing anything. Used that way it adds no lock-in. The reranker the map already chose, Qwen3-Reranker-0.6B, works like this.

## 5. Managed services

### Azure AI Search and Foundry IQ

Microsoft's [embedding model migration guide](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-migrate-embedding-models) (ms.date 2026-11-17) is explicit:

- "Changing a vectorizer or deployment name doesn't regenerate indexed vectors." To change embedding spaces, the application generates document embeddings again.
- "Equal vector dimensions don't establish compatibility between models."
- "A vector field's dimensions can't be changed after the field is created."
- With built-in indexers, "This migration path requires a full source rebuild into a new index using a new indexer." Push ingestion can add a second vector field instead.
- Costs named for a rebuild: "Full-corpus embeddings, source reads, applicable paid enrichment, and duplicate index capacity." Microsoft calls the migration approaches "ingestion architecture choices, not automatic migration features".

Model retirement forces such migrations on a schedule. The [Foundry retirement schedule](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-retirement-schedule) (ms.date 2026-11-18) lists `text-embedding-3-small`, `text-embedding-3-large` and `text-embedding-ada-002` as retiring on 2028-02-09. Cohere Embed v3 retires on 2027-02-15. The migration guide says to "Regenerate v3 document embeddings for the v5 migration", even though Cohere's v5 Fast and Pro share a space with each other.

Prices from the [Azure retail prices API](https://prices.azure.com/api/retail/prices) for Sweden Central, read 2026-11-19, per million tokens:

| Model | Global | Data Zone | Regional |
| --- | ---: | ---: | ---: |
| text-embedding-3-small | $0.020 | $0.022 | $0.025 |
| text-embedding-3-large | $0.130 | $0.143 | $0.158 |
| text-embedding-ada-002 | $0.100 | $0.110 | $0.121 |

The [quota](https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits) (ms.date 2026-08-20) for both text-embedding-3 models is 1,000,000 tokens per minute at quota Tier 1 and 15,000,000 at Tier 6. One request takes up to 2,048 inputs and 300,000 tokens.

### AWS Bedrock Knowledge Bases

The [`UpdateKnowledgeBase` API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_UpdateKnowledgeBase.html) allows changes to name, description and role only: "You can't change the `knowledgeBaseConfiguration` or `storageConfiguration` fields". The embedding model sits in `knowledgeBaseConfiguration`. A new model therefore means a new knowledge base and a new ingestion from the data source.

The managed knowledge base (generally available since June 2026, [announcement](https://aws.amazon.com/es/about-aws/whats-new/2026/06/amazon-bedrock-managed-knowledge-base/)) adds a "service-managed embedding model". The [documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-create.md) says "The service handles model selection, hosting, and scaling transparently" and "You cannot change the embedding model type after creating the knowledge base." A custom model must be 1,024 dimensions, float32. The documentation does not name the managed model or say what happens to stored vectors when AWS changes it. That gap is why the fingerprint needs a behavioral probe. The [Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/) (read 2026-11-19) lists index storage for the managed knowledge base at $5.00 per GB of raw data per month, embeddings with the managed model at $0, and the Retrieve API at $1.00 per 1,000 calls.

Prices from the [AWS price list API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonBedrock/current/region_index.json), publication 2026-11-11, per million input tokens:

| Model | Frankfurt (eu-central-1) | N. Virginia (us-east-1) |
| --- | ---: | ---: |
| Titan Text Embeddings V2, on demand | $0.20 | $0.02 |
| Titan Text Embeddings V2, batch | $0.10 | $0.01 |
| Cohere Embed English or Multilingual v3 | $0.10 | not checked |
| Cohere Embed v4 | $0.12 | not checked |

Titan V2 in Frankfurt costs ten times what it costs in N. Virginia. Bedrock throttles embedding models "by Requests Per Minute (RPM), not Tokens Per Minute (TPM)", and a Titan V2 request carries one input text ([Titan embeddings documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html)). A mid-size re-embed is therefore 20,000 requests, and its time is 20,000 divided by the RPM quota. I did not find a published default RPM for Titan V2.

### OVHcloud AI Endpoints

AI Endpoints serves embedding models through an API. It has no managed knowledge base. The catalog lists "AI Search" under agentic solutions as a lab product. Re-embedding is the app's job.

Prices from the [OVHcloud public catalog API](https://eu.api.ovh.com/v1/order/catalog/public/cloud?ovhSubsidiary=DE) (subsidiary DE, EUR, read 2026-11-19). The API gives prices in units of 10^-8 EUR per token, without VAT:

| Model | Price per million input tokens |
| --- | ---: |
| bge-m3 | EUR 0.01 |
| bge-multilingual-gemma2 | EUR 0.01 |
| Qwen3-Embedding-8B | EUR 0.10 |

The [capabilities page](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-capabilities) (updated 2026-11-10) limits authenticated use to "400 requests per minute, per Public Cloud project and per model". It does not state a batch size for embeddings or a model deprecation policy. With one chunk per request, a mid-size re-embed takes 50 minutes. With 32 chunks per request it takes under 2 minutes, if the endpoint accepts batches. I did not verify that.

## 6. Re-embedding time and cost

### Assumptions

- The mid-size project memory is the one from the bundle research: 30 MB of text, 20,000 chunks of at most 512 tokens with 10% overlap. That gives 33 MB of chunk text and 8.25 million tokens at 4 characters per token. The large one is ten times that: 300 MB of text, 200,000 chunks, 82.5 million tokens. A real repository of 300 MB of text is rare. The Keiko repository has 125 MB.
- Code and German text often tokenize at about 3 characters per token. That raises every cost and time by a third (11 and 110 million tokens).
- Query embeddings after the switch are not counted. They are small.

### On-device throughput (estimate)

No embedding run was made for this note. The machine has no embedding model, and downloading one was out of scope. Two measured prefill rates bound the estimate, because embedding a chunk does the same work as a prompt prefill without generation:

- llama.cpp's Apple Silicon table ([discussion #4167](https://github.com/ggml-org/llama.cpp/discussions/4167), build `c1d0e7a`) gives 1,589 tokens per second for Llama 2 7B in F16 at a 512-token prompt on an M5 Pro with 20 GPU cores. Llama 2 7B has about 6.5 billion non-embedding parameters. Qwen3-Embedding-0.6B has 596 million parameters, and its token embedding table holds 156 million of them (151,936 tokens times 1,024 dimensions), which leaves about 0.44 billion that do compute. Scaled by compute, that is about 23,000 tokens per second.
- The oMLX benchmark that the local evaluation research cites ([benchmark](https://omlx.ai/benchmarks/mrpdgltl)) measures 2,165 tokens per second prefill for Qwen3.6-35B-A3B in 4 bit on an M5 Pro, with about 3 billion active parameters. Scaled the same way, that is about 15,000 tokens per second.

Small models use a GPU less efficiently than large ones. I therefore plan with 5,000 to 20,000 tokens per second for Qwen3-Embedding-0.6B. A community MLX server claims 44,000 tokens per second on an M2 Max ([qwen3-embeddings-mlx](https://github.com/jakedahn/qwen3-embeddings-mlx)) without stating its method. I do not use that figure. Qwen3-Embedding-4B has about eight times the compute per token, so 600 to 2,500 tokens per second.

### Results

| Path | Rate | Mid-size (8.25M tokens) | Large (82.5M tokens) |
| --- | --- | ---: | ---: |
| Qwen3-Embedding-0.6B on M5 Pro | 5,000 to 20,000 tokens/s (estimate) | 7 to 28 min | 69 min to 4.6 h |
| Qwen3-Embedding-4B on M5 Pro | 600 to 2,500 tokens/s (estimate) | 55 min to 3.7 h | 9 to 37 h |
| Azure text-embedding-3, Tier 1 | 1M tokens per minute | 8 min | 83 min |
| Azure text-embedding-3, Tier 6 | 15M tokens per minute | under 1 min | 6 min |
| OVHcloud, 400 requests/min, 1 chunk each | about 165,000 tokens per minute | 50 min | 8.3 h |
| OVHcloud, 400 requests/min, 32 chunks each (unverified) | about 5.3M tokens per minute | 2 min | 16 min |
| Bedrock Titan V2, one text per request | RPM quota (not found) | 20,000 / RPM min | 200,000 / RPM min |

| Model and location | Mid-size | Large |
| --- | ---: | ---: |
| Azure text-embedding-3-small, Data Zone | $0.18 | $1.82 |
| Azure text-embedding-3-large, Data Zone | $1.18 | $11.80 |
| Bedrock Titan V2, Frankfurt, on demand | $1.65 | $16.50 |
| Bedrock Titan V2, Frankfurt, batch | $0.83 | $8.25 |
| Bedrock Cohere Embed v4, Frankfurt | $0.99 | $9.90 |
| Bedrock managed knowledge base, managed model | $0 for embeddings, plus re-ingestion | $0 for embeddings, plus re-ingestion |
| OVHcloud bge-m3 | EUR 0.08 | EUR 0.83 |
| OVHcloud Qwen3-Embedding-8B | EUR 0.83 | EUR 8.25 |
| On-device | electricity only | electricity only |

The money is small everywhere. The real costs of a cloud re-embed are elsewhere:

- The model policy. Under the default on-device-only policy, only the on-device row is allowed.
- The duplicate index during cutover, on Azure.
- The full re-ingestion from source, on Bedrock and on Azure with built-in indexers.

The on-device cost is time: about half an hour at worst for a mid-size project memory with the 0.6B model, and an overnight job for a large one with the 4B model.

## 7. What the bundle format and the bake-off should assume

**Bundle format.** Keep the bundle research's design: chunk text with stable chunk ids, one vector file per embedding space, reuse only on an exact fingerprint match, re-embed under the model policy, and lexical-only import as the last resort. Two additions:

1. Record the lexical analyzer in `memory.json`: tokenizer, stemmer and language, stop-word list, and BM25 `k1` and `b`. The lexical index is the only retrieval that survives without a model, so it must give the same results on every machine.
2. Add a behavioral probe to `space.json`: the vectors of a few fixed probe texts, stored in the bundle. The importer embeds the same probes and compares by cosine, not by exact hash. This catches a provider-side model change behind a stable name. Keiko does the same with a rounded hash.

Leave out vector translation, adapters, SPLADE and ColBERT token vectors. None of them removes the dependency on a model. Each adds size or a lossy step, and re-embedding from chunk text costs minutes.

**Bake-off.** The arms keep one dense model each, with BM25 as a second lane. Three measurements go into the bake-off plan:

1. Re-embedding throughput of the chosen embedding model on the M5 Pro and at the chosen cloud provider, in tokens per second over the gold project memory. This replaces the estimates above.
2. A lexical-only arm on the gold question set, split by code and prose questions. It shows how much a project memory loses when an importer has no allowed embedding model.
3. A model-swap run: re-embed the gold project memory with a second model (for example Qwen3-Embedding-4B or the provider's model), rerun the gold questions and record the time and the change in answers. This tests the claim directly.

For the managed retrieval arm (ADR-0009), record that its index cannot be exported in a form the bundle can reuse. A bundle made from it would carry chunk text and need a re-embed on import.

## Not verified

- No embedding model was run. On-device times are estimates from scaled prefill rates.
- The MTEB code tasks ask in natural language. The gap between BM25 and dense retrieval for identifier or error-message lookups was not measured.
- BM25 with a German analyzer was not tested.
- The default RPM quota for Bedrock Titan V2, and whether OVHcloud embeddings accept a batch of inputs per request.
- What Bedrock does with stored vectors when it changes its service-managed embedding model.
- Vendor-trained models such as Qwen3-Embedding and SPLADE-Code may have seen training splits of some MTEB code tasks. The MTEB numbers are the vendors' or MTEB's runs, not mine.

## Method

- Read the bundle research, ADR-0008 to ADR-0010 and the map on `origin/main`.
- Read Keiko `dev` at `bc57815` through `gh api`: `knowledge-pods.md`, the embedding governance ledger, the reindex handler, the indexing orchestrator, the gateway embedding probe and issue #4338. No local clone was used.
- Downloaded the per-task JSON results for seven models from the MTEB results repository and averaged the main scores with a short Python script. Multi-subset tasks (the CodeSearchNet variants and CodeEditSearch) are the mean over their language subsets, as MTEB computes them.
- Read prices from the Azure retail prices API, the AWS price list API and the OVHcloud catalog API on 2026-11-19, and the rules from each provider's current documentation.
- Read the papers on arXiv in their latest version. Numbers come from their tables.

## Sources

Keiko, `oscharko-dev/Keiko` at [`bc57815`](https://github.com/oscharko-dev/Keiko/tree/bc57815cbb383faa6531dd416afa5fe6ccd3d56a) on `dev`, 2026-11-19:

- [`docs/local-knowledge/knowledge-pods.md`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/local-knowledge/knowledge-pods.md)
- [`docs/local-knowledge/embedding-space-governance-ledger.md`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/docs/local-knowledge/embedding-space-governance-ledger.md)
- [`packages/keiko-server/src/local-knowledge-handlers.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-server/src/local-knowledge-handlers.ts)
- [`packages/keiko-local-knowledge/src/indexing/orchestrator.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-local-knowledge/src/indexing/orchestrator.ts)
- [`packages/keiko-model-gateway/src/embedding.ts`](https://github.com/oscharko-dev/Keiko/blob/bc57815cbb383faa6531dd416afa5fe6ccd3d56a/packages/keiko-model-gateway/src/embedding.ts)
- Issue [#4338](https://github.com/oscharko-dev/Keiko/issues/4338), "Let the person choose the embedding model when several are configured", open

Prior research in this repo:

- [Portable project memory bundle](https://github.com/Niko4417/OPT-cc/blob/research/bundle-format/docs/research/bundle-format.md), 2026-11-17
- [Memory fit](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md), 2026-11-17, for the choice of Qwen3-Embedding-0.6B
- [Local evaluation](https://github.com/Niko4417/OPT-cc/blob/research/local-evaluation/docs/research/local-evaluation.md), for the oMLX prefill benchmark

Papers:

- Jha, Zhang, Shmatikov, Morris, "Harnessing the Universal Geometry of Embeddings", [arXiv 2505.12540](https://arxiv.org/abs/2505.12540), v4 2026-01-26
- Dar, "mini-vec2vec: Scaling Universal Geometry Alignment with Linear Transformations", [arXiv 2510.02348](https://arxiv.org/abs/2510.02348), v4 2026-02-17
- Vejendla, "Drift-Adapter: A Practical Approach to Near Zero-Downtime Embedding Model Upgrades in Vector Databases", [arXiv 2509.23471](https://arxiv.org/abs/2509.23471), EMNLP 2025
- Thakur et al., "BEIR: A Heterogenous Benchmark for Zero-shot Evaluation of Information Retrieval Models", [arXiv 2104.08663](https://arxiv.org/abs/2104.08663), NeurIPS 2021
- Li et al., "CoIR: A Comprehensive Benchmark for Code Information Retrieval Models", [arXiv 2407.02883](https://arxiv.org/abs/2407.02883), ACL 2025
- Zhang et al., "Qwen3 Embedding: Advancing Text Embedding and Reranking Through Foundation Models", [arXiv 2506.05176](https://arxiv.org/abs/2506.05176), v3 2025-06-11
- Lassance et al., "SPLADE-v3: New baselines for SPLADE", [arXiv 2403.06789](https://arxiv.org/abs/2403.06789), 2024-03-11
- Lupart et al., "On the Challenges and Opportunities of Learned Sparse Retrieval for Code", [arXiv 2603.22008](https://arxiv.org/abs/2603.22008), 2026-03-23
- Santhanam et al., "ColBERTv2: Effective and Efficient Retrieval via Lightweight Late Interaction", [arXiv 2112.01488](https://arxiv.org/abs/2112.01488), NAACL 2022

Benchmark data:

- MTEB results repository at [`cde9311`](https://github.com/embeddings-benchmark/results/tree/cde931100092c418d944693da9193b012b15af38/results), 2026-11-19: `mteb__baseline-bm25s`, `naver__splade-code-06B`, `naver__splade-code-8B`, `lightonai__GTE-ModernColBERT-v1`, `Qwen__Qwen3-Embedding-0.6B`
- llama.cpp, [Performance of llama.cpp on Apple Silicon M-series](https://github.com/ggml-org/llama.cpp/discussions/4167), read 2026-11-19
- [naver/splade-code-06B model card](https://huggingface.co/naver/splade-code-06B)

Providers, all read 2026-11-19:

- Microsoft: [Embedding model migration](https://learn.microsoft.com/en-us/azure/search/vector-search-how-to-migrate-embedding-models) (ms.date 2026-11-17), [Update or rebuild an index](https://learn.microsoft.com/en-us/azure/search/search-howto-reindex) (ms.date 2026-11-09), [Model retirement schedule](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-retirement-schedule) (ms.date 2026-11-18), [Quotas and limits](https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits) (ms.date 2026-08-20), [Azure retail prices API](https://prices.azure.com/api/retail/prices)
- AWS: [UpdateKnowledgeBase](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent_UpdateKnowledgeBase.html), [Create a managed knowledge base](https://docs.aws.amazon.com/bedrock/latest/userguide/kb-managed-create.md), [Titan Text Embeddings models](https://docs.aws.amazon.com/bedrock/latest/userguide/titan-embedding-models.html), [Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), [price list API](https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonBedrock/current/region_index.json) (publication 2026-11-11)
- OVHcloud: [AI Endpoints capabilities](https://docs.ovhcloud.com/en/guides/public-cloud/ai-machine-learning/ai-endpoints-capabilities) (updated 2026-11-10), [AI Endpoints catalog](https://www.ovhcloud.com/en/public-cloud/ai-endpoints/catalog/), [public catalog API](https://eu.api.ovh.com/v1/order/catalog/public/cloud?ovhSubsidiary=DE)
