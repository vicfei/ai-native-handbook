# models/ — Model Architecture Learning Handbook

[中文](README.zh.md)

Every inference-optimization lever (batching, quantization, speculative decoding, KV management) cuts into a specific part of the model architecture. Without the structure, tuning is divination. This volume uses a **real checkpoint** (Qwen3.5-0.8B, 1.7GB, publicly downloadable) as the dissection subject, and walks the full causal chain: **structure → cost → optimization**.

## Learning path (in order)

| Chapter | Content | Questions answered |
|---|---|---|
| [01-from-transformer-to-token.md](01-from-transformer-to-token.md) | Five-step derivation from "how a word becomes numbers" to the autoregressive loop, with a hand-computed attention mini-example | What is the model actually computing? Why can't layers be skipped? |
| [02-memory-bound-and-batch.md](02-memory-bound-and-batch.md) | The 54GB bill, Roofline, and the mathematics of batching | Why is GPU inference memory-bound? Why is batching free money? |
| [03-checkpoint-anatomy-qwen3-5-0.8b.md](03-checkpoint-anatomy-qwen3-5-0.8b.md) | Per-tensor weight census (every number accounted), full data-flow diagram, the projection/gate primitives | What is every tensor in a weight file, and where does it sit in the pipeline? |
| [04-gdn-vs-full-attention.md](04-gdn-vs-full-attention.md) | Gated DeltaNet vs full attention anatomy; gated attention; the MTP×batch tension | Why 3:1 interleaving? When is speculative decoding net-negative? |

## Key takeaways

- **Every generated token requires reading the full weight set once** — the product of two inevitabilities: function composition (layer chain) and the von Neumann architecture (data lives in memory, compute lives elsewhere). Every inference optimization attacks one term of this law.
- **Parameter count and file size cross-validate**: 0.87B numbers × 2 bytes (bf16) = 1.63 GiB ≈ the actual 1.7GB file. Per-tensor census closes the accounting — the standard method for auditing any checkpoint (script in chapter 03).
- **Projection = multiply by a table = translation between spaces; gate = a learned volume knob.** In any anatomy diagram, tensor shapes are squeezed out by upstream/downstream: input side follows the residual-stream width, output side follows what the next step needs.
- **GDN state is constant (does not grow with sequence length); full-attention KV grows linearly** — the 3:1 interleave trades linear-cost throughput for exact retrieval, and is why cache parameters like vLLM's `--mamba-cache-mode` exist.

## Dissection subject

- Model: Qwen3.5-0.8B (`Qwen3_5ForConditionalGeneration`, natively multimodal, 24-layer GDN-hybrid + 12-layer ViT + MTP head)
- Tooling: any HF-format safetensors file + ~20 lines of Python (reads the file header JSON; no weight loading needed) — see chapter 03.
