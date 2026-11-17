# Memory fit of the lab models on a 64 GB Mac

Date: 2026-11-17. Ticket: issue #4. Map: issue #1.

## Question

Do `mlx-community/Qwen3.6-35B-A3B-4bit` (20.4 GB) and `mlx-community/gemma-4-31b-it-4bit` (18.4 GB) fit together on a 64 GB Mac, given that macOS gives the GPU about three quarters of RAM by default, once prompt caches, a local embedding model, a reranker, the app and its index are added? Which embedding model and reranker handle German and English code and prose within the memory left over? If the pair does not fit, what is the smallest change that makes it fit?

## Answer

The pair fits. On a 64 GB Mac (68.7 GB in decimal units) the planning total is about 62 GB at typical load, with `Qwen/Qwen3-Embedding-0.6B` as the embedding model and `Qwen/Qwen3-Reranker-0.6B` as the reranker. Both are Apache-2.0.

The margin depends on two settings. With the prompt-cache caps from Keiko's lab (4 GB for the answer model, 6 GB for the checker) fully used, the total reaches about 68.3 GB and leaves 0.4 GB. Without a prompt length limit, one 128k-token prompt per model needs 14.3 GB of KV cache, and mlx-lm does not count the active request against the cap. The smallest change is configuration only:

- cap the prompt caches at 2 GB for the answer model and 4 GB for the checker;
- limit every route's prompt to 32k tokens.

With both, the upper bound drops to about 64.3 GB, a margin of 4.4 GB. No model changes.

The GPU limit is less tight than the question assumes. On the M5 Pro with 64 GB, Metal reports a recommended working set of 55.66 GB, which is 81% of RAM, not 75%. GPU allocations total 49.1 GB at typical load and 50.8 GB at the upper bound with the smaller caps.

## Memory budget

All figures are decimal GB (10^9 bytes). "Typical" means one 32k-token prompt in flight on each model server and empty stored caches. "Upper bound" means the prompt caches are full up to their cap. The basis column says whether a figure is computed from a primary source, measured, or a planning estimate.

| Item | Typical | Upper bound, lab caps 4 + 6 GB | Upper bound, caps 2 + 4 GB | Basis |
| --- | ---: | ---: | ---: | --- |
| Answer model weights, Qwen3.6-35B-A3B-4bit | 20.40 | 20.40 | 20.40 | Safetensors size at the pinned revision |
| Checker weights, gemma-4-31b-it-4bit | 18.41 | 18.41 | 18.41 | Safetensors size at the pinned revision |
| Answer model KV cache and prompt cache | 0.74 | 4.00 | 2.00 | Computed from config.json; cap from lab.sh |
| Checker KV cache and prompt cache | 3.52 | 6.00 | 4.00 | Computed from config.json; cap from lab.sh |
| Prefill and decode working memory, both servers | 2.00 | 2.00 | 2.00 | Estimate, not measured |
| Embedding model, Qwen3-Embedding-0.6B in bf16, with a batch of 16 chunks of 1,024 tokens | 2.20 | 2.20 | 2.20 | 1.19 GB weights; 1.0 GB batch is an estimate |
| Reranker, Qwen3-Reranker-0.6B in bf16, with a batch of 8 pairs | 1.80 | 1.80 | 1.80 | 1.19 GB weights; 0.6 GB batch is an estimate |
| **GPU subtotal** | **49.07** | **54.81** | **50.81** | Limit 55.66 GB, measured |
| Vector index, keyword index and chunk text | 0.50 | 1.00 | 1.00 | Computed for 30k to 100k chunks |
| LiteLLM proxy | 0.50 | 0.50 | 0.50 | Estimate, not measured |
| App backend and UI | 2.00 | 2.00 | 2.00 | Estimate; the app is chosen in issue #10 |
| macOS and a consultant's usual apps | 10.00 | 10.00 | 10.00 | Planning assumption |
| **Total** | **62.07** | **68.31** | **64.31** | RAM is 68.72 GB |
| **Margin** | **6.65** | **0.41** | **4.41** | |

If the embedding model and reranker run on the CPU instead of Metal, their memory moves out of the GPU subtotal but stays in the total, because Apple silicon has one pool of unified memory.

## How each line was computed

### RAM and the GPU limit

The Mac in question is an Apple M5 Pro (`hw.model` Mac17,8) with `hw.memsize` 68,719,476,736 bytes, which is 64 GiB or 68.72 GB (measured with `sysctl` on 2026-11-17). Its `iogpu.wired_limit_mb` is 0, the system default.

Metal reports `recommendedMaxWorkingSetSize` = 55,662,788,608 bytes (55.66 GB, 51.84 GiB) on this machine. That is 81.0% of RAM. It was measured with a three-line Swift program that reads the property from `MTLCreateSystemDefaultDevice()`. Apple documents the property as "an approximation of how much memory, in bytes, this GPU device can allocate without affecting its runtime performance" and advises keeping the GPU footprint below it ([Apple docs](https://developer.apple.com/documentation/metal/mtldevice/recommendedmaxworkingsetsize)). The "three quarters" in Keiko's lab README is a rounded rule of thumb. On this machine it would understate the limit by 4.1 GB.

mlx-lm uses this value directly. At startup, `mlx_lm.server` calls `maybe_set_recommended_wired_limit()` ([server.py L2080](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/server.py#L2080)), which sets MLX's wired limit to `max_recommended_working_set_size` ([utils.py L74-83](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/utils.py#L74-L83)). It prints a warning only when one model's weights exceed 90% of that size ([generate.py L238-272](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/generate.py#L238-L272)). MLX documents that a wired limit above the system wired limit is an error and that `sudo sysctl iogpu.wired_limit_mb=<MB>` raises the system limit ([mx.set_wired_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_wired_limit.html)). The budget above stays under the default, so no system setting needs to change. That matches Keiko's lab, which changes no system setting.

MLX keeps freed buffers in a cache that "defaults to the memory limit" ([mx.set_cache_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_cache_limit.html)), and the memory limit defaults to 1.5 times the recommended working set ([mx.set_memory_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_memory_limit.html)). A model server therefore keeps its peak allocation after a request ends. Budget each server at its peak, not its average.

### Weights

The safetensors files at the revisions Keiko's lab pins sum to 20.40 GB for `mlx-community/Qwen3.6-35B-A3B-4bit` at `38740b847e4c` and 18.41 GB for `mlx-community/gemma-4-31b-it-4bit` at `696d436c4047` (Hugging Face tree API, both revisions are the current `main`). Loaded weights take about the file size. This is an upper bound for Qwen, because mlx-lm drops the multi-token-prediction weights at load ([qwen3_5.py L354](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/qwen3_5.py#L354)).

### KV cache per token

Both models use hybrid attention, so only a few layers grow with the prompt. All KV entries are bf16 (2 bytes).

**Answer model.** The [config.json](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit/blob/38740b847e4cb78f352aba30aa41c76e08e6eb46/config.json) has 40 layers with `full_attention_interval` 4, so 10 full-attention layers and 30 linear-attention layers. The full-attention layers have 2 KV heads of dimension 256. mlx-lm gives them a `KVCache` and the linear layers an `ArraysCache` of fixed size ([qwen3_5.py L347-348](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/qwen3_5.py#L347-L348)).

- Growing part: 10 layers x 2 (keys and values) x 2 heads x 256 x 2 bytes = 20,480 bytes, or 20 KiB per token.
- Fixed part: each linear layer keeps a float32 state of 32 value heads x 128 x 128 ([gated_delta.py L583](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gated_delta.py#L583)) plus a convolution state of 3 x 8,192 values. Over 30 layers that is 0.064 GB per sequence.

**Checker.** The [config.json](https://huggingface.co/mlx-community/gemma-4-31b-it-4bit/blob/696d436c404745a59f30e4939a658162b0a9e57f/config.json) has 60 layers: 10 `full_attention` and 50 `sliding_attention` with a 1,024-token window. Full layers use `num_global_key_value_heads` 4 and `global_head_dim` 512. Sliding layers use 16 KV heads of dimension 256. mlx-lm uses the global head dimension for full layers ([gemma4_text.py L187-193](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gemma4_text.py#L187-L193)). With `attention_k_eq_v` the full layers have no value projection, but the cache still stores keys and values as separate tensors after different normalization ([gemma4_text.py L253-271](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gemma4_text.py#L253-L271)). Sliding layers get a `RotatingKVCache` capped at the window ([gemma4_text.py L677-690](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gemma4_text.py#L677-L690)).

- Growing part: 10 layers x 2 x 4 heads x 512 x 2 bytes = 81,920 bytes, or 80 KiB per token. That is four times the answer model.
- Fixed part: 50 layers x 1,024 tokens x 2 x 16 heads x 256 x 2 bytes = 0.84 GB, up to 1.26 GB while a 512-token prefill step is added to a full window.

KV cache for one sequence, in GB:

| Prompt tokens | Answer model | Checker | Both |
| ---: | ---: | ---: | ---: |
| 8,192 | 0.23 | 1.51 | 1.74 |
| 16,384 | 0.40 | 2.18 | 2.58 |
| 32,768 | 0.74 | 3.52 | 4.26 |
| 65,536 | 1.41 | 6.21 | 7.61 |
| 131,072 | 2.75 | 11.58 | 14.33 |
| 262,144 (the configured maximum) | 5.43 | 22.31 | 27.75 |

A RAG prompt with a system prompt, 10 to 20 retrieved chunks of 500 tokens, the question and a short history is 8k to 16k tokens. The checker also sees the answer and its cited sources. 32k tokens covers both with room to spare, so the budget uses 32k.

### Prompt caches

Keiko's lab starts each server with `--prompt-cache-size 2 --prompt-cache-bytes 4GB` for the answer model and `6GB` for the checker, plus `--prefill-step-size 512` and one request at a time (Keiko's local `scripts/testing/review-quality/lab/lab.sh`, function `serve` and its two calls, lines 188-216). mlx-lm parses "GB" as 10^9 bytes ([utils.py `_parse_size`](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/utils.py#L86-L92)).

The cap covers stored caches only. After each request is admitted, the server trims stored caches to `prompt_cache_bytes` minus the bytes of the active request ([server.py L886-889](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/server.py#L886-L889)), and `trim_to` clamps a negative budget to zero ([cache.py L1717-1732](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/cache.py#L1717-L1732)). A request larger than the cap still runs at full size. So the cap bounds a server's KV memory only if prompts are limited too. At 32k tokens the active requests need 0.74 GB and 3.52 GB, below both the lab caps and the smaller 2 GB and 4 GB caps. At 128k tokens they need 2.75 GB and 11.58 GB, and the checker alone exceeds its 6 GB cap by 5.6 GB.

### Embedding model and reranker

`Qwen/Qwen3-Embedding-0.6B` and `Qwen/Qwen3-Reranker-0.6B` each have 595,776,512 bf16 parameters, 1.19 GB of safetensors ([embedding card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B), [reranker card](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B), Hugging Face API). Both configs have 28 layers, hidden size 1,024, 8 KV heads of dimension 128. The embedding model accepts 32,768 tokens and the reranker 40,960.

The batch allowances (1.0 GB and 0.6 GB) are estimates. An encoder-style forward pass without a cache holds one layer's activations at a time. Chunks of up to 1,024 tokens in batches of 16, and reranker batches of 8 query-chunk pairs, keep that small. Do not send whole files of 32k tokens to the embedding model in large batches. Measure both on the target machine in the build phase.

### Vector index

Assume a mid-size modernization project with 300,000 lines of code, 1,500 Markdown files and 3,000 issues and pull requests. At about 10 tokens per code line, 2,000 tokens per Markdown file and 1,000 tokens per issue or pull request, that is about 9 million tokens. In chunks of 400 tokens with overlap, that gives 25,000 to 30,000 chunks.

- Vectors: 30,000 x 1,024 dimensions x 4 bytes (float32) = 0.12 GB. At 100,000 chunks it is 0.41 GB.
- Graph links of an HNSW index at 16 neighbors add roughly 128 bytes per vector (estimate), under 0.02 GB.
- Chunk text and a keyword index add a few hundred MB at most.

The budget uses 0.5 GB typical and 1.0 GB upper. The index is small next to the models, so its size does not decide the fit.

### LiteLLM, app and OS

These three lines are planning estimates. No measurement on the target machine exists yet. The app line is 2 GB because the stack is not yet chosen (issue #10). The OS line is 10 GB because consultants keep a browser, a chat client and an editor open while they use the project memory.

## Evidence from Keiko's lab

Keiko's lab (`scripts/testing/review-quality/lab/README.md` and `lab.sh` in the local Keiko checkout) runs this exact 4-bit pair at the same pinned revisions on an M5 Pro with 64 GB:

- It runs both models as two `mlx_lm.server` processes (mlx-lm 0.32.2) behind LiteLLM, together with Keiko's review gate.
- Before a run it waits until macOS can reclaim 42 GiB (45.1 GB, since `reclaimable_gb` divides by 2^30). That is 6.3 GB above the 38.8 GB of weights, which is less than the 10 GB of prompt-cache caps. The lab therefore expects the caches not to fill at once, or macOS to free more memory during the run.
- The 30-PR development run takes about 3.6 hours and the seed run about 1 hour at 4 bit.
- The gateway declares a 131,072-token context window for both models. At that length the checker's active KV cache alone would be 11.6 GB, so a project memory should set a lower prompt limit than the lab.

This shows that the pair loads and serves for hours on the target hardware. It does not show swap use or memory pressure, because the lab's evidence files are body-free counts and durations. The lab also runs no embedding model, reranker or app.

## Embedding model and reranker choice

**Embedding model: `Qwen/Qwen3-Embedding-0.6B`** (Apache-2.0, 1.19 GB in bf16, 1,024 dimensions, 32k context).

German results from the MTEB results repository ([Qwen3-Embedding-0.6B](https://github.com/embeddings-benchmark/results/tree/main/results/Qwen__Qwen3-Embedding-0.6B/b22da495047858cce924d27d76261e96be6febc0), [bge-m3](https://github.com/embeddings-benchmark/results/tree/main/results/BAAI__bge-m3/5617a9f61b028005a4858fdac845db406aefb181), [multilingual-e5-large-instruct](https://github.com/embeddings-benchmark/results/tree/main/results/intfloat__multilingual-e5-large-instruct/baa7be480a7de1539afce709c8f13f833a510e0a), [Qwen3-Embedding-4B](https://github.com/embeddings-benchmark/results/tree/main/results/Qwen__Qwen3-Embedding-4B/636cd9bf47d976946cdbb2b0c3ca0cb2f8eea5ff)), main score x 100, test split, German subset:

| Task | Qwen3-Emb-0.6B | bge-m3 | mE5-large-instruct | Qwen3-Emb-4B |
| --- | ---: | ---: | ---: | ---: |
| German1Retrieval | 95.34 | 92.45 | 95.77 | 96.52 |
| GermanHealthcare1Retrieval | 71.35 | 72.48 | 74.24 | 84.80 |
| GermanLegal1Retrieval | 61.80 | 60.89 | 64.10 | 70.47 |
| WikipediaRetrievalMultilingual (de) | 89.03 | 90.28 | 92.55 | 92.32 |
| MLQARetrieval (deu-deu) | 66.36 | 70.04 | 70.92 | 74.11 |
| BelebeleRetrieval (deu-deu) | 93.21 | 93.91 | 94.33 | 96.06 |
| GermanSTSBenchmark | 83.99 | 80.79 | 84.84 | 90.03 |
| AppsRetrieval (code) | 75.34 | 14.74 | 34.89 | 89.18 |
| StackOverflowQA (code) | 89.99 | 80.60 | 85.82 | 94.32 |

On German prose the three 0.6B-class models are within about 4 points of each other, with multilingual-e5-large-instruct slightly ahead. On code, Qwen3-Embedding-0.6B leads by 40 to 60 points on AppsRetrieval. multilingual-e5-large-instruct is MIT-licensed but has `max_position_embeddings` 514, so it truncates at 512 tokens ([config](https://huggingface.co/intfloat/multilingual-e5-large-instruct/blob/main/config.json)). bge-m3 is MIT with an 8,192-token limit ([card](https://huggingface.co/BAAI/bge-m3)). Qwen3-Embedding-0.6B is the only one of the three that is strong on both code and German prose. Qwen also reports it at 64.33 on MMTEB against 59.56 for bge-m3 and 63.22 for multilingual-e5-large-instruct ([card](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B), vendor-reported).

`Qwen/Qwen3-Embedding-4B` (Apache-2.0) beats the 0.6B model on every German task in the table, by 13 points on GermanHealthcare1Retrieval and 9 points on GermanLegal1Retrieval, and by 14 points on AppsRetrieval. In bf16 it needs 8.04 GB (4.02 billion parameters), 6.85 GB more than the 0.6B model. That breaks the upper bound even with the smaller caps (71.2 GB). An 8-bit build would add about 3.1 GB and leave about 1.3 GB margin, but no source checked here measures retrieval quality after quantizing it. Keep it as the upgrade to test in the evaluation, not the default. Switching changes the dimension from 1,024 to 2,560, so it needs a full re-index. At this index size that costs embedding time, not memory.

**Reranker: `Qwen/Qwen3-Reranker-0.6B`** (Apache-2.0, 1.19 GB in bf16, 40k context).

Qwen reports it at 66.36 on the MMTEB retrieval subset and 73.42 on MTEB-Code, against 58.36 and 41.38 for `BAAI/bge-reranker-v2-m3` (Apache-2.0, 0.57 billion float32 parameters) ([reranker card](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B), vendor-reported). No independent German reranking result exists for either model in the MTEB results repository, so the German quality of the reranker is unverified. The evaluation on the public test dataset must cover it. It shares the tokenizer and base model family with the embedding model, so one runtime can serve both.

Excluded for licensing: `jinaai/jina-embeddings-v3` and `jinaai/jina-reranker-v2-base-multilingual` are CC BY-NC 4.0 (Hugging Face API). `google/embeddinggemma-300m` uses the Gemma terms, which need a separate review.

## If memory still runs short

In order of size of change:

1. Set `--prompt-cache-bytes 2GB` on the answer model and `4GB` on the checker, and limit prompts to 32k tokens in LiteLLM or the app. This saves 4 GB at the upper bound. It is the recommended default, not a fallback.
2. Lower the checker cap further. At 32k prompts the checker's active request needs 3.52 GB. A lower cap evicts stored prompts sooner and costs prefill time on repeated prefixes, not correctness.
3. Try `--kv-bits 8` on the checker. mlx-lm documents it as reducing memory for long contexts and serving one request at a time ([server.py L2054](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/server.py#L2054)), which a single-user install already does. Its effect on Gemma 4's rotating sliding-window cache was not checked.
4. Load the checker only while it checks an answer. This frees 18.4 GB between checks but adds a load time per check that has not been measured.

## Not verified

- No model was run for this note. The prefill working memory, the embedding and reranker batch memory, LiteLLM, the app and the OS lines are estimates. The build phase should measure peak memory per process with `vm_stat` and the MLX memory counters under a 32k-token prompt.
- German quality of the reranker. Only vendor-reported, mostly English and Chinese, numbers exist.
- Swap or memory pressure in Keiko's lab runs.

## Sources

- Model configs at the pinned revisions: [Qwen3.6-35B-A3B-4bit config.json](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit/blob/38740b847e4cb78f352aba30aa41c76e08e6eb46/config.json), [gemma-4-31b-it-4bit config.json](https://huggingface.co/mlx-community/gemma-4-31b-it-4bit/blob/696d436c404745a59f30e4939a658162b0a9e57f/config.json).
- mlx-lm v0.32.2 source: [server.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/server.py), [utils.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/utils.py), [generate.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/generate.py), [models/cache.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/cache.py), [models/qwen3_5.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/qwen3_5.py), [models/gemma4_text.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gemma4_text.py), [models/gated_delta.py](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/mlx_lm/models/gated_delta.py), [README "Large Models"](https://github.com/ml-explore/mlx-lm/blob/v0.32.2/README.md#large-models).
- MLX docs: [set_wired_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_wired_limit.html), [set_memory_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_memory_limit.html), [set_cache_limit](https://ml-explore.github.io/mlx/build/html/python/_autosummary/mlx.core.set_cache_limit.html).
- Apple: [MTLDevice.recommendedMaxWorkingSetSize](https://developer.apple.com/documentation/metal/mtldevice/recommendedmaxworkingsetsize).
- Model cards: [Qwen3-Embedding-0.6B](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B), [Qwen3-Embedding-4B](https://huggingface.co/Qwen/Qwen3-Embedding-4B), [Qwen3-Reranker-0.6B](https://huggingface.co/Qwen/Qwen3-Reranker-0.6B), [bge-m3](https://huggingface.co/BAAI/bge-m3), [bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3), [multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct).
- MTEB per-task results: [embeddings-benchmark/results](https://github.com/embeddings-benchmark/results).
- Keiko's lab, local checkout: `/Users/niko/code/Keiko/scripts/testing/review-quality/lab/README.md` and `lab.sh`.
- Measurements on the M5 Pro with 64 GB, 2026-11-17: `sysctl hw.memsize hw.model iogpu.wired_limit_mb` and `MTLCreateSystemDefaultDevice().recommendedMaxWorkingSetSize`.
