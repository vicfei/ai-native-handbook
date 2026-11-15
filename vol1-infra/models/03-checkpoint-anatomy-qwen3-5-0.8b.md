# 03 · Checkpoint Anatomy: a Per-Tensor Census of Qwen3.5-0.8B

[中文](03-checkpoint-anatomy-qwen3-5-0.8b.zh.md)

> Methodology in one line: **a safetensors file header is a JSON manifest — reading it requires no weight loading.** A tensor's parameter count = the product of its shape dimensions; sum over the file for the total; parameter count × dtype bytes should match the file size. A census only counts if the books close.

## The census script (~20 lines, works on any HF model)

```python
import json, struct, math
f = open('model.safetensors-00001-of-00001.safetensors','rb')
n = struct.unpack('<Q', f.read(8))[0]          # header length (first 8 bytes, little-endian)
hdr = json.loads(f.read(n)); hdr.pop('__metadata__', None)
numel = lambda k: math.prod(hdr[k]['shape'])   # product of shape dims
total = sum(numel(k) for k in hdr)
print(f"{total:,} params ≈ {total/1e9:.2f}B → bf16 ×2B = {total*2/2**30:.2f} GiB")
```

## The ledger (0.87B, cross-validated against the 1.7GB file ✓)

| Component | Source of the number | Parameters |
|---|---|---|
| Embedding (tied, doubles as LM head) | 248,320 × 1024 | 254,279,680 |
| Language trunk, 24 layers | 18×21,555,360 (GDN) + 6×18,352,640 (full attn) | 498,112,320 |
| final norm | 1024 | 1,024 |
| Vision tower (12-layer ViT + merger) | below | 100,592,896 |
| MTP head | below | 20,452,864 |
| **Total** | | **873,438,784 ≈ 0.87B → 1.63 GiB** |

Structure: `Qwen3_5ForConditionalGeneration` = **ViT tower** (12 layers × 768 hidden, patch 16, merger projects to 1024 to align with text) + **language trunk** (24 layers, 18 GDN + 6 full-attention interleaved 3:1 — full attention at indices 3/7/11/15/19/23) + **tied embedding** + **MTP side head**. Code lineage (vLLM `qwen3_5.py`): vision front end inherits Qwen3-VL; language skeleton inherits Qwen3-Next (the GDN pioneer).

## GDN layer ledger (18 identical layers)

| Tensor | Shape | Params | Role |
|---|---|---|---|
| in_proj_qkv | (6144, 1024) | 6,291,456 | q‖k‖v fused (each 16 heads × 128 = 2048, ×3 = 6144) |
| mlp.gate/up/down | 3×(3584, 1024) | 11,010,048 | the SwiGLU trio |
| in_proj_z | (2048, 1024) | 2,097,152 | raw material of output gate z |
| out_proj | (1024, 2048) | 2,097,152 | 16 heads × 128 back to trunk width 1024 |
| conv1d | (6144, 1, 4) | 24,576 | depthwise short-range conv (4-wide window per channel) |
| in_proj_a / in_proj_b | (16, 1024) each | 32,768 | per-head input for the forget gate / write gate |
| A_log / dt_bias | (16,) | 32 | per-head decay baseline (log-parameterized) and write bias |
| norms ×4 | — | ~2.3K | normalizations |
| **per-layer total** | | **21,555,360** | |

## Full-attention layer ledger (6 identical layers)

q_proj (4096,1024)=4,194,304 (**gated: q 2048 + gate 2048**) + k/v_proj (512,1024)×2=1,048,576 (GQA 8Q:2KV × head_dim 256) + o_proj (1024,2048)=2,097,152 + QK-norm 512 + two layer norms 2,048 + MLP 11,010,048 = **18,352,640**.

Evidence chain for gated attention: q_proj output = 8 heads × 256 × **(1 + attn_output_gate)** = 4096, and vLLM source `torch.chunk(q_gate, 2)` splits q and gate — the extra 2048 is a per-head output gate, not a 16th head.

## Vision tower and MTP

- **Vision tower** (100.6M): patch_embed conv (768, 3, 2, 16, 16) — 3 color channels × 2-frame temporal window × 16×16 pixel kernel; 12 ViT blocks (7.1M each: fused qkv + MLP + norms, **with biases** — unlike the bias-free language side); the merger concatenates 2×2 neighboring patches (3072 = 768×4) and projects to 1024.
- **MTP** (20.5M): `fc` (1024,2048) fuses [token embedding; trunk-final hidden state] (each pre-normalized) → one layer **spec-identical to a trunk full-attention layer** → norm. It reuses the trunk embedding/LM head — 20M for a free "forecaster" (the speculative-decoding drafter).

## The full data-flow diagram (GDN layer)

```
x (1024) ─┬─ in_proj_qkv(6144) → conv1d(w=4) → split 16 heads × 128 → q,k through norm
          │   α = σ(in_proj_a(x) − exp(A_log))        (per-head forget gate)
          │   β = σ(in_proj_b(x) + dt_bias)           (per-head write gate)
          │   recurrence: k̃=k−Sᵀq; S←α·S+β·v·k̃ᵀ; h=Sᵀq  (S: 128×128 per head, constant size)
          │   h concat 16 heads → 2048
          ├─ in_proj_z(2048) → z
          │   output = out_proj(1024,2048)( SiLU(z) ⊙ h )
          └─ MLP (SwiGLU 3584) → residual → norm → next layer
```

## Two primitives + one shape law (the keys to any anatomy diagram)

1. **Projection = multiply by a table = translation between spaces.** Every `_proj` suffix does this: in_proj_z (2048,1024) reads "translate x from 1024 to z in 2048".
2. **Gate = a learned volume knob.** A 0~1 value (sigmoid) or soft switch (SiLU) multiplied onto a signal: z, SwiGLU's gate, α/β, and full attention's q_gate are one primitive at different installation points.
3. **The shape law**: a projection's input side = upstream width (residual stream 1024); output side = downstream need (16 heads × 128 = 2048; three-in-one = 6144; one knob per head = 16). **Shapes are not hyperparameters — they are squeezed out by neighbors.**
