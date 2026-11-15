# Pod 取证——四个推理 Pod 的原始配置捕获

[English](README.md)

[../8xh20-four-pod-reference.zh.md](../8xh20-four-pod-reference.zh.md) 所述四生产 pod 的原始配置取证。

**出处：** 2026-10-28 自云平台控制台与 pod 终端捕获（`/proc/1/environ`、`cmdline`、实弹探针）——即**进程当时真实在跑的东西**，比部署清单更强的证据（env 可能与 YAML 漂移）。性质为终端输出转写、经 OCR 校对；**非** `kubectl get -o yaml` 导出。

## 文件

| 文件 | Pod | 看点 |
|---|---|---|
| `ivs-llm-chat.md` | 旗舰 chat（DeepSeek-V4-Flash，4 卡） | vLLM 完整命令行与引擎旗标 |
| `ivs-llm-chat-lite.md` | 高并发 lite（Qwen3.8-27B，2 卡） | MTP×2、`--mamba-cache-mode align` |
| `ivs-llm-multi.md` | 检索双件（1 卡） | **含 MODELS"残条"事故现场**——重复首条逐字保留 |
| `ivs-llm-file-parser.md` | 文档解析 | 三容器时代 + VLM 档实弹测试记录 |
| `deployment-file-parser-ACTIVE-20261109.yaml` | 解析 pod 实际生效的替换清单 | 双容器形态 |
| `deployment-file-parser-413-replace.yaml` | 替换草案 | 评审史保留 |
| `deployment-mineru-vllm-413.yaml` | 候选清单 | — |
| `prod-file-parser-pod-20261106.yaml` | pod 快照 | — |

## 脱敏说明

镜像仓库域名与集群名泛化；一处节点 IP 以占位符替换。模型名、参数、镜像 tag 与事故现场逐字保留。
