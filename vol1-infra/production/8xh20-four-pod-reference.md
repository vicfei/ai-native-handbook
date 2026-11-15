# Production Reference: One 8×H20 Node, Four Inference Pods — the Full Configuration Record

[中文](8xh20-four-pod-reference.zh.md)

> Real production configuration from an offline private cloud (owner-approved disclosure: genuine parameters kept; company name, internal domains, and credentials excluded). The four services cover the four canonical shapes of LLM serving: flagship chat, lightweight high-concurrency, document parsing, and the retrieval pair.

## 1. Cluster topology and constraints

- One node, 8× H20 (96GB HBM3 each); fully offline (models and images distributed internally)
- The platform allocates whole GPUs per pod: each pod gets 1/2/4/8 cards, total 8
- Allocation: **4+2+1+1** — and under "keep all four services + the granularity constraint" it is essentially the only solution: the flagship (a large MoE, ~50GiB/card of weights at TP4) physically cannot fit on 2 cards → locked at 4; every service needs ≥1 card, consuming the remaining 4

## 2. The four pods, parameter by parameter

### ivs-llm-chat — flagship chat (DeepSeek-V4-Flash, 4 GPUs)

| Parameter | Value |
|---|---|
| Engine | vLLM 0.25.1 (vllm-docker line) |
| Parallelism | TP4 + EP4 |
| Weights | DeepSeek-V4-Flash (factory FP8) on NAS, separate from the image |
| Context / concurrency | max-model-len 196608 / max-num-seqs 32 |
| KV cache | fp8, block-size 256 |
| Memory | gpu-memory-utilization 0.9; native KV offloading 64GB |
| Speculative | dspark ×5 (greedy) |
| Probes | **full trio present** (startup 30×30s / readiness 15×3 / liveness 60×3, all tcpSocket 8000) — contrasting with lite's liveness-only; evidence of no shared baseline template. Also startup budget 30min < engine's own 1h wait: internally inconsistent |
| Misc | VLLM_ENGINE_READY_TIMEOUT_S=3600; four routing aliases (**command-line structure suspect**: per the export, 1 alias value + 3 stray positionals + a non-upstream flag `--enable-max-think`; verify via /v1/models) |

### ivs-llm-chat-lite — lightweight high-concurrency (Qwen3.8-27B, 2 GPUs)

| Parameter | Value |
|---|---|
| Parallelism / precision | TP2, BF16 unquantized |
| Context / concurrency | max-model-len 131072 / max-num-seqs 96 (3× the flagship — the high-concurrency/short-request stance) |
| KV cache | fp8_e4m3, block-size 64 |
| Memory | gpu-memory-utilization 0.92 + PYTORCH_ALLOC_CONF expandable_segments (fragmentation control) |
| Speculative | MTP ×2 (shipped with the checkpoint) |
| Hybrid-arch companion | `--mamba-cache-mode align` (state-cache mode for the GDN linear-attention layers) |
| Tool calling | qwen3_coder parser + auto-tool-choice |
| Probes | liveness only (initialDelay 600s) — audit defect #1 |

### ivs-llm-multi — the retrieval pair (Embedding + Reranker, 1 GPU)

A single container hosting two instances: a launcher process reads the `MODELS` env var and starts two vLLM servers (vllm-dynamic 0.10.1 line). The corrected configuration:

```yaml
- name: MODELS
  value: |
    [
      { "name": "embedding",
        "command": ["python3","-m","vllm.entrypoints.openai.api_server",
          "--model","/data/models/Qwen3-Embedding-4B",
          "--served-model-name","qwen3-embedding-4b","embedding",
          "--host","0.0.0.0","--port","8002",
          "--max_model_len","8192","--max_num_batched_tokens","16384",
          "--gpu-memory-utilization","0.40",
          "--task","embedding"] },
      { "name": "reranker",
        "command": ["python3","-m","vllm.entrypoints.openai.api_server",
          "--model","/data/models/Qwen3-Reranker-4B",
          "--served-model-name","qwen3-reranker-4b","reranker",
          "--host","0.0.0.0","--port","8003",
          "--max_model_len","8192","--max_num_batched_tokens","8192",
          "--gpu-memory-utilization","0.28","--enforce-eager",
          "--task","score",
          "--hf_overrides","{\"architectures\":[\"Qwen3ForSequenceClassification\"],\"classifier_from_token\":[\"no\",\"yes\"],\"is_original_qwen3_reranker\":true}"] }
    ]
```

Three load-bearing semantics (each paid for by an incident):

1. **The reranker must run `--task score` + the hf_overrides trio**: the loader subtracts the LM-head row of "no" from that of "yes" to build a single-row discriminative head (mathematically identical to the official yes/no softmax), and `/v1/rerank` returns true discrimination probabilities. Started with `--task embedding` (the old config), scores degrade to cosine similarity between hidden states never trained for contrast — the endpoint answers, ranking appears to work, and the semantics are wrong.
2. **The embedding instance's `--task embedding` must never be dropped**: without it the engine defaults to generate by architecture → `/v1/embeddings` returns 404 → upstream ingestion and recall silently break (the port still listens; the health endpoint still returns 200).
3. util 0.40/0.28 + batch 8192 + enforce-eager is the OOM-repair band: activation peaks of large rerank requests once blew through the margin at 0.45+0.45 with batch 16384 (a runtime death that went unnoticed for 10 hours).

### ivs-llm-file-parser — document parsing (1 GPU)

- Two containers: the parsing main container (vision pipeline, 4.1.3 line, **weights baked into the image** — the opposite of the other three pods' NAS-separated route; every upgrade rebuilds and redistributes a multi-GB image) plus a format-conversion container
- Memory semantics: automatic tier detection (no explicit mem-fraction); the API generation moved from a synchronous `/file_parse` single endpoint to an async task API (V1 contract incompatible — callers must adapt)

## 3. Six defect classes found by the audit (a configuration-governance reference)

| # | Defect | Root cause | Fix direction |
|---|---|---|---|
| 1 | liveness only; no readiness/startup | cold-start duration guessed via an initialDelay constant → rolling-update traffic black hole + slow-load crash loop | the probe trio |
| 2 | util 0.90~0.92 at the ceiling, no load tests | decisions without evidence chains; activation peaks can break through | stepped load tests + explicit max-num-batched-tokens |
| 3 | MTP×2 alongside 96 slots | two parameters describing opposite service shapes | stage by acceptance×concurrency curve / adaptive verification |
| 4 | 96×128K capacity illusion | nominal product far exceeds the KV pool | recompute at "concurrency × average context" + queue caps |
| 5 | retrieval task semantic mismatch | a task flag's semantics fork; endpoint OK ≠ semantics OK | semantic acceptance into the release checklist |
| 6 | engine-version vs architecture-generation mismatch | new architecture on an old engine; upstream fixes never arrived | regression-checklisted upgrades |

The common root cause: **configuration delivered without an evidence chain**. Attaching load-test curves, semantic acceptance, and forensics to every fix is the shortest path from "template delivery" to "auditable operations".
