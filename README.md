<div align="center">

# AI-Native Handbook

[![License: MIT](https://img.shields.io/badge/License-MIT-lightgrey.svg)](LICENSE)
[![Last commit](https://img.shields.io/github/last-commit/vicfei/ai-native-handbook)](https://github.com/vicfei/ai-native-handbook/commits)

**The operating manual for AI-native engineering teams — written from a live 8×H20 production cluster, a real 0.8B checkpoint dissected tensor by tensor, and incidents that actually happened.**

[中文](README.zh-CN.md) · English

</div>

---

Three volumes, one shared foundation, three roles:

| Volume | For | Contents | Status |
|---|---|---|---|
| [vol1-infra/](vol1-infra/) | Infra / inference engineers | **Model-structure handbook** (checkpoint dissection, Roofline accounting, GDN hybrid anatomy) · quantization toolchain & benchmarks · vLLM deployment recipes · kernel learning · **production reference: a real 4-pod / 8×H20 inference cluster, parameter by parameter** | ✅ live |
| [vol2-agent-dev/](vol2-agent-dev/) | Agent application developers | Enterprise RAG (open-source platform forks, Java-side integration, production deployment) · enterprise Harness (deployment & plugin development) · application scenarios | 🚧 skeleton |
| vol3-org/ (later) | Managers / transformation leads | Evaluation-driven management, automation levels | 📋 planned |

**What makes this handbook different:**

- **A production cluster, parameter by parameter** — one 8×H20 (96GB) node serving four real workloads: flagship chat (TP4+EP, 196K context), high-concurrency lite (TP2, MTP×2, 96 slots), document parsing, and an embedding+reranker pair. Full configuration record plus the six defect classes an audit found — including a silent semantic failure most teams would never notice ([vol1-infra/production/](vol1-infra/production/)).
- **A real checkpoint, tensor by tensor** — a per-tensor weight census of Qwen3.5-0.8B where the books close (0.87B × 2B = 1.63 GiB ≈ the actual file), used to derive the one law behind every inference optimization ([vol1-infra/models/](vol1-infra/models/)).
- **Two environments, labeled honestly** — production content validated on the 8×H20 cluster; lab experiments run on an RTX 4060 8GB laptop, a deliberately tight bench that forces you to account for every megabyte of video memory.

**Where to start:** [vol1-infra/models/](vol1-infra/models/) — four chapters deriving, from first principles and a real 0.8B checkpoint, why *every generated token requires reading the full weight set once*, and why that single law explains batching, quantization, MoE, and speculative decoding. It is the shared foundation for everything else in this handbook.

**Flagship production content:** [vol1-infra/production/8xh20-four-pod-reference.md](vol1-infra/production/8xh20-four-pod-reference.md) — the complete configuration record of a 4-pod inference cluster on one 8×H20 node (flagship chat / high-concurrency lite / document parsing / the embedding+reranker pair), including the six defect classes an audit found and the semantic traps (e.g. why `--task embedding` vs `--task score` on the same reranker model is the difference between cosine similarity and true discrimination).

> Companion map: [awesome-ai-native](https://github.com/vicfei/awesome-ai-native) — the thesis-driven map of the AI-native enterprise (why and where-to). This handbook is the *how*.

## Conventions

- English is the primary language; every chapter has a `.zh.md` Chinese companion (language-switch links at the top of each file).
- All content is first-hand: real checkpoints, real clusters, real incidents; every number is reproducible (the census script is ~20 lines of Python).
- Publishing rules in [PUBLISHING.md](PUBLISHING.md) — short version: technical detail fully open (owner-approved); company names and credentials never appear.

## Provenance

Production side: an offline private cloud, one 8×H20 node, four inference pods — parameters, incident timelines, and fixes all from live systems. Lab side: RTX 4060 Laptop 8GB (WSL2), vLLM v0.29, llmcompressor 0.14, lm-eval — where the constraint *is* the curriculum.
