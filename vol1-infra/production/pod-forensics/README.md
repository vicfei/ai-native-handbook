# Pod Forensics — original captures of the four inference pods

[中文](README.zh.md)

Raw configuration forensics for the four production pods described in [../8xh20-four-pod-reference.md](../8xh20-four-pod-reference.md).

**Provenance:** captured 2026-10-28 from the cloud-platform console and pod terminals (`/proc/1/environ`, `cmdline`, live probes) — i.e., *what the processes were actually running*, which is stronger evidence than the deployment manifest (env can drift from YAML). These are terminal-output transcriptions, OCR cross-checked; **not** `kubectl get -o yaml` exports.

## Files

| File | Pod | Notable |
|---|---|---|
| `ivs-llm-chat.md` | flagship chat (DeepSeek-V4-Flash, 4 GPUs) | full vLLM cmdline, all engine flags |
| `ivs-llm-chat-lite.md` | high-concurrency lite (Qwen3.8-27B, 2 GPUs) | MTP ×2, `--mamba-cache-mode align` |
| `ivs-llm-multi.md` | retrieval pair (1 GPU) | **contains the MODELS env "ghost entry" incident scene** — the duplicate first entry, verbatim |
| `ivs-llm-file-parser.md` | document parser | 3-container era, VLM backend live-test records |
| `deployment-file-parser-ACTIVE-20261109.yaml` | parser, the live replacement manifest | 2-container form actually deployed |
| `deployment-file-parser-413-replace.yaml` | parser replacement draft | review history preserved |
| `deployment-mineru-vllm-413.yaml` | parser candidate manifest | — |
| `prod-file-parser-pod-20261106.yaml` | parser pod snapshot | — |

## Sanitization

Registry domains and cluster names are genericized; one node IP replaced by a placeholder. Model names, parameters, image tags, and incident scenes are kept verbatim.
