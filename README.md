<div align="center">

# AI-Native Handbook

[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey.svg)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/vicfei/ai-native-handbook)](https://github.com/vicfei/ai-native-handbook/commits)

**Field notes from the frontier of LLM serving — a production 8×H20 inference cluster,
hybrid linear-attention architectures, and the discipline of auditable operations.**

[中文](README.zh-CN.md) · English

</div>

---

## What this is

The operating manual for AI-native engineering teams, in three volumes. Everything here is **first-hand**: production parameters, incident forensics, and reproducible measurements — not summaries of other people's blog posts.

**Why it's different:**

- **A production cluster, parameter by parameter.** One 8×H20 (96GB HBM3) node serving four real workloads — flagship chat (TP4+EP, 196K context, speculative decoding ×5), high-concurrency lite (TP2, MTP ×2, 96 slots), multimodal document parsing, and a cross-encoder retrieval pair. The complete configuration record, plus the **six defect classes** a systematic audit found — including a silent semantic failure most teams would never notice ([vol1-infra/production/](vol1-infra/production/)).
- **Architecture-first foundations.** Four chapters derive, from first principles up to a real production-family checkpoint, the one law behind every inference optimization — *every generated token requires reading the full weight set once* — and use it to unify batching, quantization, MoE, and speculative decoding. A per-tensor census methodology that makes the parameter accounting close to the byte ([vol1-infra/models/](vol1-infra/models/)).
- **Evidence-chain operations.** Load-test curves, semantic acceptance tests, and incident forensics attached to every configuration claim — the audit methodology itself is content, not just the fixes.

Auxiliary lab experiments (quantization matrices, kernel studies, cache hit-rate benchmarks) live inside the chapters where they apply, clearly scoped and labeled.

## The three volumes

| Volume | For | Contents | Status |
|---|---|---|---|
| [vol1-infra/](vol1-infra/) | Infra / inference engineers | Model-structure foundations (Roofline accounting, hybrid GDN vs full-attention anatomy, gated attention) · quantization toolchain & benchmarks · vLLM deployment recipes · kernel learning · **production cluster reference** | ✅ live |
| [vol2-agent-dev/](vol2-agent-dev/) | Agent application developers | Enterprise RAG (platform forks, Java-side integration, production deployment) · enterprise harness (deployment & plugin development) · application scenarios | 🚧 skeleton |
| vol3-org/ (later) | Managers / transformation leads | Evaluation-driven management, automation levels | 📋 planned |

**Where to start:** [vol1-infra/models/](vol1-infra/models/) — the four-chapter foundation. It builds everything from first principles with no prerequisites (the on-ramp for newcomers), yet lands squarely on production concerns: why hybrid linear-attention models need dual state management, when speculative decoding turns net-negative, and what a tensor shape tells you about its role.

> Companion map: [awesome-ai-native](https://github.com/vicfei/awesome-ai-native) — the thesis-driven map of the AI-native enterprise (the *why* and *where-to*). This handbook is the *how*.

## Conventions

- English is the primary language; every chapter has a `.zh.md` Chinese companion (language-switch links at the top of each file).
- Production content and auxiliary experiments are labeled as such where they appear; hardware and version specifics stay inside the chapters that use them.
- Publishing rules in [PUBLISHING.md](PUBLISHING.md).
