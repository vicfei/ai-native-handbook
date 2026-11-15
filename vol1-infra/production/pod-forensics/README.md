# Pod Forensics — original captures of the four inference pods

[中文](README.zh.md)

Raw configuration forensics for the four production pods described in [../8xh20-four-pod-reference.md](../8xh20-four-pod-reference.md).

**Provenance:** captured 2026-10-28 from the cloud-platform console and pod terminals (`/proc/1/environ`, `cmdline`, live probes) — i.e., *what the processes were actually running*, which is stronger evidence than the deployment manifest (env can drift from YAML). These are terminal-output transcriptions, OCR cross-checked; **not** `kubectl get -o yaml` exports.

## Layout

**Top level — one manifest per pod (the initial configurations):**

| File | Pod | Form |
|---|---|---|
| `ivs-llm-chat.yaml` | flagship chat (4 GPUs) | **authentic** deployment export, 2026-11-09 |
| `ivs-llm-chat-lite.yaml` | high-concurrency lite (2 GPUs) | **authentic** deployment export, 2026-11-09 |
| `ivs-llm-multi.yaml` | retrieval pair (1 GPU) | **authentic** deployment export, 2026-11-09 (MODELS env as live) |
| `ivs-llm-file-parser.yaml` | document parser (1 GPU) | **authentic** pod snapshot, original 3-container era (2026-11-06) |

**`raw/` — the evidence layer:** the four verbatim forensic captures (env/cmdline/probes, incl. the MODELS ghost-entry scene) plus the authentic file-parser manifests from the 2026-11-09 replacement (draft, candidate, and the live ACTIVE version), and the superseded reconstructions (`*.reconstructed.yaml`) authored before the authentic exports arrived.

## Sanitization

Registry domains and cluster names are genericized; one node IP replaced by a placeholder. Model names, parameters, image tags, and incident scenes are kept verbatim.
