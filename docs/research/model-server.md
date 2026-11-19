# Local model server and LiteLLM setup

Date: 2026-11-17. Ticket: [#3](https://github.com/Niko4417/OPT-cc/issues/3), part of map [#1](https://github.com/Niko4417/OPT-cc/issues/1).

## Question

Which native model server should run the Keiko for Quality lab models (`mlx-community/Qwen3.6-35B-A3B-4bit` as the answer model, `mlx-community/gemma-4-31b-it-4bit` as the checker) on the Mac: `mlx_lm.server`, Ollama with its MLX engine, or llama.cpp? Which of them also runs on Linux and Windows with NVIDIA GPUs? How do embeddings and reranking get served locally behind the same LiteLLM proxy? How does a LiteLLM route declare its model location (on-device, client-hosted, third-party cloud) so that the app can read it?

## Answer

1. **Run the two chat models with `mlx_lm.server` on the Mac, one process per model, as Keiko's review lab does.** It is the only one of the three servers that loads the two pinned mlx-community builds as they are. Keiko's lab ran exactly these builds with mlx-lm 0.32.2 behind LiteLLM 1.106.6 ([lab README](#sources), K1). Ollama runs MLX only for its own library tags or an experimental import, and its importer has an open bug for checkpoints with per-layer quantization overrides, which the Qwen3.6 build has. llama.cpp does not read MLX weights at all.
2. **Serve embeddings and reranking with llama.cpp's `llama-server`.** `mlx_lm.server` has no embeddings or rerank route, and Ollama has no rerank route. `llama-server` has both (`--embedding`, `--reranking`), runs on Metal and CUDA, and ships Windows and Linux binaries. Use small GGUF models such as `Qwen/Qwen3-Embedding-0.6B-GGUF` and `ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF` (639 MB each, Apache-2.0). The model choice belongs to ticket #4.
3. **LiteLLM routes both to the local servers.** Embeddings use the `hosted_vllm/` (or `openai/`) provider with an `api_base` ending in `/v1`. Rerank uses the `hosted_vllm/` provider: LiteLLM posts `query`, `documents` and `top_n` to `<api_base>/rerank` and reads `results[].index` and `results[].relevance_score`, which is the Jina-format answer of `llama-server`'s `/rerank`. This pairing is read from source code on both sides and is not yet tested end to end.
4. **Declare the model location as a custom `model_info` key on every route,** for example `model_location: on-device`. LiteLLM's `ModelInfo` type accepts extra keys, and `GET /model/info` (also `/v1/model/info`) returns `model_info` from `config.yaml` merged with LiteLLM's own data. The app reads the field and applies the project memory's model policy. Keiko already discovers route limits this way.
5. **On Linux and Windows with NVIDIA the MLX builds do not run.** Keep the LiteLLM route names and swap the server and weights: GGUF builds of the same two models on `llama-server` (or Ollama's GGUF tags), or vLLM on a client server. LiteLLM hides the swap from the app. The weights then differ from the Mac's, so evaluation results do not carry over one to one.

## 1. What Keiko's two labs show

Keiko has two working labs that put a native model server behind LiteLLM.

**The review lab** (`scripts/testing/review-quality/lab/lab.sh`) serves exactly the pair this project wants. It pins `mlx-community/Qwen3.6-35B-A3B-4bit` at revision `38740b847e4c` (20.4 GB) and `mlx-community/gemma-4-31b-it-4bit` at `696d436c4047` (18.4 GB), installs `mlx-lm==0.32.2` and `litellm[proxy]==1.106.6` into one Python virtual environment, and starts one `mlx_lm.server` per model on 127.0.0.1 ports 8080 and 8081, with LiteLLM on 127.0.0.1:4000 (K1, K2). Each server runs with `--max-tokens 8192 --prompt-cache-size 2 --prompt-cache-bytes <4 or 6 GB> --decode-concurrency 1 --prompt-concurrency 1 --prefill-step-size 512` plus the sampling settings and `--chat-template-args` per model (K2, lines 188 to 195). The LiteLLM route for each model is `model: openai/<local snapshot path>`, `api_base: http://127.0.0.1:<port>/v1`, a dummy `api_key`, and `model_info` with `max_input_tokens: 131072`, `max_output_tokens: 8192` and `supports_function_calling: true` (K2, lines 67 to 97). The README records one weakness: "A model server whose generation thread dies answers every later request with 404", so `lab.sh` runs a watchdog that restarts the pair (K1).

**The Coding Workbench lab** (`docs/qa/coding-workbench-lab/`) used Ollama 0.35 with its MLX engine and the Ollama library tag `gemma4:31b-mlx`, wrapped in a Modelfile that pins `num_ctx 131072`. LiteLLM v1.104.0 reached it through `hosted_vllm/keiko-gemma4-31b` at `http://host.docker.internal:11434/v1` (K3, K4). That lab ran LiteLLM in Docker, which this project avoids. Its route config documents the `model_info` declarations Keiko reads through `/model/info`: `context_window`, `max_output_tokens`, `supports_function_calling` and `supports_vision` (K4). Keiko's gateway guide adds that a route without declared limits stays on a 4,096-token placeholder until a probe proves more (K5).

So both server options work behind LiteLLM. Only the review lab ran the exact mlx-community builds.

## 2. The three servers on the Mac with the pinned builds

| | `mlx_lm.server` | Ollama (MLX engine) | llama.cpp `llama-server` |
| --- | --- | --- | --- |
| License, latest release | MIT, v0.32.2 (2026-10-16) | MIT, v0.40.16 (2026-11-09) | MIT, v1.0.0 (2026-11-13) |
| Loads the two pinned mlx-community builds | Yes, proven by Keiko's lab | Only through experimental import; open bug for the Qwen3.6 build | No, GGUF only |
| Same models in its own format | Not applicable | `qwen3.6:35b-a3b-nvfp4` (23 GB), `gemma4:31b-mlx` (19 GB) | `ggml-org/Qwen3.6-35B-A3B-GGUF`, `ggml-org/gemma-4-31B-it-GGUF` |
| Chat routes | `/v1/chat/completions`, `/chat/completions`, `/v1/completions`, `/v1/models`, `/health` | OpenAI routes plus `/v1/responses` | OpenAI and Anthropic routes |
| Embeddings | No | Yes, `/v1/embeddings` | Yes, `--embedding` |
| Rerank | No | No | Yes, `--reranking` |
| Models per process | One | Many, loaded on demand | One, or many in router mode |
| Linux and Windows with NVIDIA | No (see section 3) | Yes, with GGUF tags | Yes, CUDA builds |

**`mlx_lm.server`.** The server's POST routes are `/v1/completions`, `/v1/chat/completions` and `/chat/completions`, and its GET routes are `/v1/models` and `/health` (M1, lines 1248 to 1256 and 1779 to 1781). It binds to 127.0.0.1:8080 by default (M1, lines 1925 and 1931) and parses tool calls (M1, line 77). Release notes add a Gemma 4 tool call parser in v0.31.2 and parallel tool call fixes in v0.31.3 (M3). The model code has `qwen3_5_moe.py` and `gemma4.py` (M4). Both pinned builds are mlx-vlm conversions: the Qwen3.6 card says it was converted with mlx-vlm 0.4.4, and the Gemma card names an mlx-vlm source checkout (H1, H2). Their configs declare `Qwen3_5MoeForConditionalGeneration` and `Gemma4ForConditionalGeneration` (H3). mlx-lm loads them as text models, as Keiko's lab shows (K1). The project's own documentation says "The MLX LM server is not recommended for production as it only implements basic security checks" (M2). For a single-user install bound to loopback behind LiteLLM, that is acceptable.

**Ollama.** Ollama 0.19 (2026-03-30) introduced MLX on Apple silicon as a preview (O1). In v0.40.0 (2026-09-25), "model architectures supported by the MLX runtime will automatically run on MLX", and the release names gemma4, qwen3.6 and qwen3.5 (O2). That applies to Ollama's own library tags. The library has `qwen3.6:35b-a3b-nvfp4` (MLX, 23 GB) and `gemma4:31b-mlx` (MLX, 19 GB) (O3, O4). These are different quantizations from the mlx-community builds (NVFP4 or Ollama's own MLX build instead of affine 4-bit, group size 64), so they are not the lab's weights. To load an mlx-community checkpoint, a user runs `ollama create <name> --experimental -f Modelfile` with `FROM <snapshot dir>` (O5). The Qwen3.6 build declares 4-bit affine quantization with 80 per-layer 8-bit overrides on the router gates (H3: `language_model.model.layers.N.mlp.gate` and `...shared_expert_gate`). Ollama issue #18789 (open, reported on 0.35.1) shows that an import with exactly this kind of override succeeds, but the model then fails to load with a `quantized_matmul` shape mismatch (O5). Other open issues report mlx-community Gemma 4 checkpoints that import but do not load (#18540, #16740) (O6). I did not test the two pinned builds on Ollama. The import path is experimental and has known failures for this checkpoint shape.

**llama.cpp.** `llama-server` reads GGUF files (L1). It cannot load the MLX safetensors builds. GGUF builds of both models exist from the ggml-org account (`ggml-org/Qwen3.6-35B-A3B-GGUF`, `ggml-org/gemma-4-31B-it-GGUF`) and from Google (`google/gemma-4-31B-it-qat-q4_0-gguf`) (H4). Using them means different weights from the lab pair.

## 3. Linux and Windows with NVIDIA

- **mlx-lm.** Its README describes it as a package for "generating text and fine-tuning large language models on Apple silicon with MLX" and names no other platform (M5). MLX itself has a CUDA backend for Linux (`pip install mlx[cuda12]` or `mlx[cuda13]`, NVIDIA SM 7.5 or newer, glibc 2.35 or newer) and no Windows build (M6). mlx-lm does not document running on that backend, so treat it as unsupported.
- **Ollama.** Ollama ships Linux and Windows builds, and its release notes cover NVIDIA CUDA (v0.40.16 adds CUDA 13.5 for Windows on ARM) (O7). Its MLX engine is a different matter. In October 2026 an Ollama contributor wrote that MLX support is "not yet available on Linux _and Windows_", and on 2026-11-07 a user reported that MLX-only tags still fail on 0.40.15 for Linux (O8). Release assets named `ollama-linux-amd64-mlx` and `ollama-windows-amd64-mlx` exist at least since v0.32.5 (O7), but I found no release note that documents MLX models running there. On NVIDIA, Ollama runs the GGUF tags (`qwen3.6:35b-a3b-q4_K_M`, 24 GB; `gemma4:31b-it-q4_K_M`, 20 GB) (O3, O4).
- **llama.cpp.** It builds with Metal on macOS and with CUDA on Linux and Windows (L2), and its release page ships prebuilt binaries plus an installer script for both (L1). The server also has an official CUDA container image (L3).
- **vLLM** is the usual server for a client-hosted model on NVIDIA. Keiko's Coding Workbench lab treats it as the customer's server shape and reaches it through the same `hosted_vllm/` LiteLLM provider (K3).

Portability therefore lives at the LiteLLM layer, not the model server. The app always calls the same LiteLLM route names. Only the `litellm_params` of each route change per platform.

## 4. Embeddings and reranking behind LiteLLM

**Which local servers have the routes.** `mlx_lm.server` has neither route (M1). Ollama has `/v1/embeddings` but documents no rerank endpoint (O9), and an open feature request asks for first-class embedding and reranker support on the MLX path (#16076) (O6). `llama-server` has both:

- `--embedding` restricts a server to an embedding model, and `--pooling {none,mean,cls,last,rank}` sets the pooling. It serves the OpenAI-compatible `POST /v1/embeddings` (L3). The Qwen3-Embedding GGUF card gives `llama-server -m model.gguf --embedding --pooling last -ub 8192` (H5).
- `--reranking` enables `POST /reranking` with the aliases `/rerank`, `/v1/rerank` and `/v1/reranking`. A body with `query` and `documents` gets the Jina format back: `model`, `object`, `usage` and `results` with `{index, relevance_score}`, sorted by score and cut to `top_n`. A body with `texts` gets the TEI format instead (L3).
- In router mode, one `llama-server` without `-m` loads and unloads several models on demand, picked by the `model` field of each request (L3). Two plain processes on two ports are simpler to watch from a start script.

Hugging Face's text-embeddings-inference (Apache-2.0, v1.9.4) is a second option for both routes. It runs on Metal through Homebrew and on CUDA, and serves `/embed` and `/rerank` (T1). It adds a separate Rust toolchain or Homebrew package, and its rerank body uses `texts`, so LiteLLM would need its `huggingface` rerank provider rather than `hosted_vllm`.

**How LiteLLM routes them.**

- The proxy exposes `/embeddings`, `/v1/embeddings`, `/rerank` and `/v1/rerank` (S1, `litellm/proxy/_types.py` lines 266 and 330 to 331). LiteLLM's rerank endpoint is Cohere-compatible (S2).
- Embeddings: LiteLLM documents the `openai/` prefix plus `api_base` for "calling `/embedding` endpoints on OpenAI Compatible Servers" (S3), and the `hosted_vllm/` provider has its own embedding module (S1, `litellm/llms/hosted_vllm/embedding/`). The `ollama/` provider calls Ollama's native `/api/embed` (S1, `litellm/llms/ollama/completion/handler.py`).
- Rerank: the providers in LiteLLM's rerank table are Cohere, Together AI, Azure AI, Jina AI, Bedrock, HuggingFace, Infinity, vLLM, DeepInfra, Vertex AI, Fireworks AI, Voyage and watsonx.ai (S2). There is no llama.cpp or Ollama rerank provider. The `hosted_vllm` rerank transformer builds the URL as `<api_base>/rerank` (it rewrites a trailing `/v1/rerank` to `/rerank`), sends `query`, `documents`, `top_n` and `return_documents`, and requires `index` and `relevance_score` in each result (S1, `litellm/llms/hosted_vllm/rerank/transformation.py` lines 56 to 70, 156 to 180 and 215 to 246). That matches `llama-server`'s Jina-format reply. LiteLLM's vLLM docs show the rerank route as `model: hosted_vllm/<name>` with `api_base: http://localhost:8000` (S4).

Not verified: I did not run this pairing. `return_documents` is not a documented `llama-server` option (L3 lists `top_n`, `model` and `return_text`), so the server may ignore it or reject it. A prototype ticket should send one rerank and one embedding request through LiteLLM to `llama-server`.

## 5. Declaring the model location on a LiteLLM route

**`model_info` custom keys.** LiteLLM's docs say: "You can attach arbitrary `model_info` fields when you create or update a model, and they pass straight through to `GET /model/info`" (S5). The source confirms it for `config.yaml` too. The `ModelInfo` Pydantic model is declared with `extra="allow"` (S1, `litellm/types/router.py` line 389). The `/model/info` handler, also mounted at `/v1/model/info`, starts from the `model_info` of `config.yaml` and adds LiteLLM's cost-map data only for keys the config does not set (S1, `litellm/proxy/proxy_server.py` lines 19053 to 19062 and 17384 to 17428). API keys are masked in the response (S5). A route therefore carries the location like this, and the app reads it next to `context_window` and the other limits it already needs:

```yaml
model_list:
  - model_name: answer
    litellm_params:
      model: openai/<snapshot path of mlx-community/Qwen3.6-35B-A3B-4bit>
      api_base: http://127.0.0.1:8080/v1
      api_key: mlx-local
    model_info:
      model_location: on-device        # on-device | client-hosted | third-party-cloud
      mode: chat
      max_input_tokens: 131072
      max_output_tokens: 8192
      supports_function_calling: true
```

**Tags.** `litellm_params.tags` exists for tag-based routing (S1, `litellm/types/router.py` line 670). With `router_settings.enable_tag_filtering: true`, a request that carries `tags` or an `x-litellm-tags` header goes only to deployments with a matching tag, and an untagged request goes to deployments tagged `default` (S6). Tag routing chooses among deployments of a model name. It is not a policy check across all routes. A tag such as `on-device` may still be useful as a second guard on a fallback route.

**Keys.** LiteLLM virtual keys can be limited to a list of models (`models` on `/key/generate`), but they need a Postgres database (S7). A single-user, local-first install has no reason to run Postgres for this.

**Recommendation for this field.** Use `model_info.model_location` with the three glossary values as the single source of truth. The app fetches `/model/info` at startup, refuses any route whose location is beyond the project memory's model policy, and fails closed on a route without the field. That keeps the policy check in the app, where the model policy lives, and works the same way for chat, embedding and rerank routes.

## 6. Recommended process layout on the Mac

| Process | Command (sketch) | Port | LiteLLM route |
| --- | --- | --- | --- |
| Answer model | `mlx_lm.server --host 127.0.0.1 --port 8080 --model <Qwen3.6 snapshot> ...` | 8080 | `openai/<path>`, `api_base .../v1` |
| Checker model | `mlx_lm.server --host 127.0.0.1 --port 8081 --model <Gemma 4 snapshot> ...` | 8081 | `openai/<path>`, `api_base .../v1` |
| Embeddings | `llama-server --host 127.0.0.1 --port 8090 -m <embedding.gguf> --embedding --pooling last` | 8090 | `hosted_vllm/<name>`, `api_base .../v1` |
| Reranker | `llama-server --host 127.0.0.1 --port 8091 -m <reranker.gguf> --reranking` | 8091 | `hosted_vllm/<name>`, `api_base http://127.0.0.1:8091` |
| LiteLLM | `litellm --config <config.yaml> --host 127.0.0.1 --port 4000` | 4000 | Not applicable |

- Take the `mlx_lm.server` flags, the pinned revisions, the loopback-only binding, the generated master key and the watchdog from Keiko's `lab.sh` (K1, K2). Every server, including `llama-server`, defaults to or is set to 127.0.0.1 (M1; L3 `--host`, default 127.0.0.1).
- Install mlx-lm and LiteLLM into one Python virtual environment, as `lab.sh` does. Install `llama-server` from the llama.cpp release binaries or Homebrew. None of this needs Docker.
- Pin versions in the start script. Keiko's lab used mlx-lm 0.32.2 and LiteLLM 1.106.6. The latest releases on 2026-11-17 are mlx-lm 0.32.2 and LiteLLM v1.109.1.
- Whether both chat models plus the embedding model and reranker fit into 64 GB is ticket #4. Keiko's lab waits for 42 GB of reclaimable memory before it starts the 4-bit pair with prompt caches of 4 GB and 6 GB (K1, K2).
- On Linux or Windows with NVIDIA, replace the two `mlx_lm.server` processes with `llama-server` on the GGUF builds (or vLLM on a client server), keep the embedding and rerank processes, and keep the LiteLLM route names.

Ollama stays a fallback for the chat models if `mlx_lm.server` turns out too fragile: its own `qwen3.6:35b-a3b-nvfp4` and `gemma4:31b-mlx` tags run on MLX and need no import, at the cost of different weights than the lab pair.

## Open points

- Run one embedding and one rerank request through LiteLLM's `hosted_vllm` provider to `llama-server` (section 4). This fits the prototype ticket.
- Decide whether the evaluation must use the identical mlx-community weights. If not, Ollama's MLX tags or llama.cpp GGUF builds become options for the chat models too, and one server could serve all four models.
- Check LiteLLM's tag filtering semantics in the router source if the app wants a second guard beyond `model_info.model_location`. I read the docs only (S6).

## Sources

Keiko (a local checkout):

- K1: `scripts/testing/review-quality/lab/README.md`
- K2: `scripts/testing/review-quality/lab/lab.sh` (lines 28 to 97 and 188 to 226)
- K3: `docs/qa/coding-workbench-lab/README.md` (sections "Deployment shape", "1. Serve the model", "2. Point LiteLLM at the model server")
- K4: `docs/qa/coding-workbench-lab/litellm.config.template.yaml`
- K5: `docs/troubleshooting/litellm-production-gateway.md` (lines 17 to 45 and 1058 to 1090)

mlx-lm and MLX:

- M1: [`mlx_lm/server.py`](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/server.py) at main on 2026-11-17
- M2: [`mlx_lm/SERVER.md`](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md)
- M3: [mlx-lm releases](https://github.com/ml-explore/mlx-lm/releases)
- M4: [`mlx_lm/models/`](https://github.com/ml-explore/mlx-lm/tree/main/mlx_lm/models)
- M5: [mlx-lm README](https://github.com/ml-explore/mlx-lm)
- M6: [MLX installation](https://ml-explore.github.io/mlx/build/html/install.html)

Hugging Face model repositories:

- H1: [mlx-community/Qwen3.6-35B-A3B-4bit](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit)
- H2: [mlx-community/gemma-4-31b-it-4bit](https://huggingface.co/mlx-community/gemma-4-31b-it-4bit)
- H3: `config.json` of both repositories at main ([Qwen3.6](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit/blob/main/config.json), [Gemma 4](https://huggingface.co/mlx-community/gemma-4-31b-it-4bit/blob/main/config.json))
- H4: [ggml-org/Qwen3.6-35B-A3B-GGUF](https://huggingface.co/ggml-org/Qwen3.6-35B-A3B-GGUF), [ggml-org/gemma-4-31B-it-GGUF](https://huggingface.co/ggml-org/gemma-4-31B-it-GGUF), [google/gemma-4-31B-it-qat-q4_0-gguf](https://huggingface.co/google/gemma-4-31B-it-qat-q4_0-gguf)
- H5: [Qwen/Qwen3-Embedding-0.6B-GGUF](https://huggingface.co/Qwen/Qwen3-Embedding-0.6B-GGUF), [ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF](https://huggingface.co/ggml-org/Qwen3-Reranker-0.6B-Q8_0-GGUF)

Ollama:

- O1: [Ollama is now powered by MLX on Apple Silicon in preview](https://ollama.com/blog/mlx), 2026-03-30
- O2: [Ollama v0.40.0 release](https://github.com/ollama/ollama/releases/tag/v0.40.0)
- O3: [qwen3.6 tags](https://ollama.com/library/qwen3.6/tags)
- O4: [gemma4 tags](https://ollama.com/library/gemma4/tags)
- O5: [ollama/ollama#18789](https://github.com/ollama/ollama/issues/18789), per-layer quantization overrides ignored on MLX import
- O6: [#18540](https://github.com/ollama/ollama/issues/18540), [#16740](https://github.com/ollama/ollama/issues/16740), [#16076](https://github.com/ollama/ollama/issues/16076)
- O7: [Ollama releases](https://github.com/ollama/ollama/releases), assets of v0.40.16 and v0.32.5
- O8: [ollama/ollama#18825](https://github.com/ollama/ollama/issues/18825)
- O9: [Ollama OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)

llama.cpp:

- L1: [llama.cpp README](https://github.com/ggml-org/llama.cpp) and [v1.0.0 release](https://github.com/ggml-org/llama.cpp/releases/tag/v1.0.0), 2026-11-13
- L2: [`docs/build.md`](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md), sections "Metal Build" and "CUDA"
- L3: [`tools/server/README.md`](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md), parameter table, `/reranking`, `/v1/embeddings`, router mode and Docker sections

LiteLLM:

- S1: [BerriAI/litellm](https://github.com/BerriAI/litellm) source at commit `7b88d94d` (2026-11-17); file paths and lines as cited
- S2: [Rerank](https://docs.litellm.ai/docs/rerank)
- S3: [Supported embedding models](https://docs.litellm.ai/docs/embedding/supported_embedding)
- S4: [vLLM provider](https://docs.litellm.ai/docs/providers/vllm)
- S5: [Model management](https://docs.litellm.ai/docs/proxy/model_management)
- S6: [Tag based routing](https://docs.litellm.ai/docs/proxy/tag_routing)
- S7: [Virtual keys](https://docs.litellm.ai/docs/proxy/virtual_keys)
- License: [LICENSE](https://github.com/BerriAI/litellm/blob/main/LICENSE), MIT outside `litellm/enterprise/`

Text embeddings inference:

- T1: [huggingface/text-embeddings-inference](https://github.com/huggingface/text-embeddings-inference) README, v1.9.4
