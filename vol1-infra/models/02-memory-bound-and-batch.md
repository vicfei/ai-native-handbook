# 02 · Memory-Bound, Roofline, and the Mathematics of Batching

[中文](02-memory-bound-and-batch.zh.md)

> The foundational law: **to generate each token, the GPU must read the entire model weight set from memory once.** This chapter derives why it holds, what it costs, and which term of the law each mainstream optimization attacks.

## The delivery-station model

- **HBM memory** = warehouse shelves (numbers live here); **compute units** = workers at benches; connected by the **transport lane** (memory bandwidth).
- **Every forward pass = one delivery run that must pick through the entire warehouse** (every weight used once) → fixed loading cost.
- **Batching = carpooling**: one run carries 96 parcels; the loading cost is amortized 96×.

## Two inevitabilities (the law's composition)

1. **Structural** (function composition): mathematically chained layers (chapter 01) → every pass must traverse every table;
2. **Physical** (von Neumann): data lives in memory, arithmetic happens in compute units — **numbers don't move, computation doesn't happen**. A number must be transported before it can multiply. Worker-side registers are only KB~MB; 54GB does not fit.

## The arithmetic: how idle are the workers? (27B model, 1 token)

```
Transport: 54 GB ÷ 4 TB/s (H20 bandwidth)    ≈ 13.5 ms   ← per-token floor time
Compute:   ~54 GFLOP ÷ 148 TFLOPS (H20)      ≈ 0.37 ms
Worker utilization ≈ 0.37 / 13.5 ≈ 3%
```

This is **memory-bound**, quantitatively: the bottleneck is transport, not work.

## Roofline: the first tool for locating bottlenecks

**Arithmetic intensity** = FLOPs ÷ bytes moved. Each weight number is "transported 2 bytes, used in 2 FLOPs" (a multiply and an add), so single-request decoding has intensity exactly **1 FLOP/B**.

The hardware **balance point** = compute ÷ bandwidth = 148 ÷ 4 = **37 FLOP/B**:

- intensity < 37 → material waits for workers (memory-bound)
- intensity > 37 → workers wait for material (compute-bound)

**The mathematics of batching**: at batch=96 each weight number is still transported once but multiplied 96 times → intensity ×96 = 96 > 37 → **the bottleneck flips**. Carpooling didn't transport less; each parcel of material just did 96 jobs.

## The optimization toolbox vs the law

| Technique | Plain-language essence | Attacks |
|---|---|---|
| **Continuous batching** | carry many parcels per run; amortize loading | structural cost amortization |
| **Quantization (fp8/int8/int4)** | shrink each parcel | physical transport volume |
| **MoE** | partition the warehouse; each token visits only needed aisles | structural "every table must be read" |
| **MTP / speculative decoding** | one run brings back 3 tokens to verify | number of fixed-cost runs |
| **KV cache** | don't recompute history notes (saves activation recompute, not weight reads) | preprocessing redundancy |

## Extension: why MTP wins at low concurrency and loses at high

MTP's ledger: **saves bandwidth (fewer runs), spends compute (draft generation + verification; rejected drafts are wasted work)**.

| | Low concurrency (few runs, idle workers) | High concurrency (full runs, busy workers) |
|---|---|---|
| Saves (bandwidth) | hits the bottleneck ✅ | irrelevant (bandwidth no longer the bottleneck) |
| Spends (compute) | workers idle — waste is free ✅ | competes with real parcels ❌ |
| Net | big win | possibly negative |

**Configuration lesson**: when `max-num-seqs` (a high-concurrency stance) and speculative decoding (a low-batch optimum) appear in the same config, two opposing service shapes are being mixed. The right fix: measure the acceptance-rate × concurrency curve and stage the setting (speculative on under low load, off/low at saturation) — or let the engine's adaptive verification decide in real time.
