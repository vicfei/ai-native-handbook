# 04 · GDN vs Full Attention: Anatomy of a Hybrid Architecture

[中文](04-gdn-vs-full-attention.zh.md)

> The Qwen3.5 / Qwen3-Next generation builds language trunks as **18 GDN layers + 6 full-attention layers (3:1 interleaved)**. This chapter contrasts the two mechanisms, costs, and roles — and explains the serving parameters they dictate.

## What each layer is

**Full attention (softmax attention)** — the classic: each token attends over all previous tokens via `softmax(QKᵀ/√d)·V`. Strongest retrieval (exact recall at any position), at quadratic compute and linearly growing KV cache. This model adds a modern refinement: **gated attention** (a per-head output gate hidden inside q_proj — see [chapter 03](03-checkpoint-anatomy-qwen3-5-0.8b.md)).

**GDN (Gated DeltaNet, linear attention)** — replaces "look at all history" with "maintain a compressed memory": a 128×128 state matrix S per head, updated recurrently per token:

```
k̃ₜ = kₜ − Sₜ₋₁ᵀ qₜ          (delta rule: subtract what's already memorized before writing — no double-counting)
Sₜ = αₜ·Sₜ₋₁ + βₜ·vₜ·k̃ₜᵀ    (α = forget gate, β = write gate; one scalar per head each)
hₜ = Sₜᵀ qₜ                   (readout)
```

Intuition: S is a 128×128 memory room; each token "discards a little old, writes a little new". **The room size never changes** — the essential difference from a KV cache.

## Cost comparison

| | GDN ×18 | Full attention ×6 |
|---|---|---|
| Sequence complexity | O(L) | O(L²) |
| State | constant ~0.5MB per layer per sequence (the S matrices) | KV per token per layer (2 KV heads × 256 dims → ~1KB/token/layer at fp8) |
| Retrieval | recent + lossy compressed memory | exact recall at any position |
| Prefill engineering | chunked-parallel kernels | standard attention backends |
| Decode engineering | fused recurrent kernels | standard attention |

**Design intent of 3:1**: 18 GDN layers carry long-sequence throughput at linear cost; every 4th layer is a full-attention "exact retrieval anchor". This is also why vLLM needs `--mamba-cache-mode` for this family — the engine manages two state forms at once (GDN's constant state + full attention's KV).

## GDN's companion parts (position and purpose)

| Part | Position | Purpose |
|---|---|---|
| conv1d (6144,1,4) | after qkv projection, before head split | recurrence can't see the immediately preceding words (they're compressed into S); this 4-wide depthwise window is a per-position peephole — a Mamba-lineage staple |
| in_proj_a/b (16,1024) | parallel to qkv, both read x | one scalar per head → α forget gate / β write gate (a manager per memory room) |
| A_log / dt_bias (16,) | inside α/β's sigmoid | per-head decay baseline (log-space training stability) and write bias |
| in_proj_z → SiLU(z)⊙h | after readout, before out_proj | output gate: per-dimension volume control over what's read from memory |

## The MTP head: speculative decoding's vehicle

MTP (Multi-Token Prediction) = a 20M side module co-trained with the trunk (structure in chapter 03) that predicts the second-next token from the same hidden state. At inference it is the drafter: guess N tokens, the trunk verifies them in one batched pass (verification is nearly free — the weights were going to be read anyway; see [chapter 02](02-memory-bound-and-batch.md)).

**The ledger**: saves bandwidth (fewer passes), spends compute (draft + verify; rejected drafts are pure waste). The win region is low batch (Qwen3.5-0.8B measured MTP acceptance length ≈ 1.8 — roughly 0.8 free tokens per step); at high batch, batching has already amortized bandwidth, and draft compute competes with real tokens.

**Configuration criterion**: `num_speculative_tokens` is not "bigger is better" — follow the acceptance-rate × concurrency curve: on at low load, off at saturation, or delegate to the engine's adaptive verification.

## Closing picture

```
sequence → [GDN]×3 → [full attn]×1 → [GDN]×3 → [full attn]×1 → … (×6 groups)
            ↑ linear-throughput layers        ↑ exact-retrieval anchors
          constant state 0.5MB/layer      KV grows linearly with tokens
```

The whole point of the hybrid: **trade 18 layers of lossy memory for linear long-sequence cost, and keep exact recall alive through 6 full-attention layers.** Once this trade is understood, the family's serving parameters (mamba-cache-mode, KV dtype, speculation level) all gain structural explanations.
