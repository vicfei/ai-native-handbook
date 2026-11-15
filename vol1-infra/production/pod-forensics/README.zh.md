# Pod 取证——四个推理 Pod 的原始配置捕获

[English](README.md)

[../8xh20-four-pod-reference.zh.md](../8xh20-four-pod-reference.zh.md) 所述四生产 pod 的原始配置取证。

**出处：** 2026-10-28 自云平台控制台与 pod 终端捕获（`/proc/1/environ`、`cmdline`、实弹探针）——即**进程当时真实在跑的东西**，比部署清单更强的证据（env 可能与 YAML 漂移）。性质为终端输出转写、经 OCR 校对；**非** `kubectl get -o yaml` 导出。

## 目录结构

**顶层——每个 pod 一份清单（最初配置）：**

| 文件 | Pod | 形态 |
|---|---|---|
| `ivs-llm-chat.yaml` | 旗舰 chat（4 卡） | Deployment，自实弹 cmdline 取证重建 |
| `ivs-llm-chat-lite.yaml` | 高并发 lite（2 卡） | Deployment，自实弹 cmdline 取证重建 |
| `ivs-llm-multi.yaml` | 检索双件（1 卡） | Deployment + 原始 MODELS env（发现时缺陷就地注释） |
| `ivs-llm-file-parser.yaml` | 文档解析（1 卡） | **真件** pod 快照，最初三容器形态（2026-11-06） |

**`raw/`——证据层：** 四份逐字取证（env/cmdline/探针，含 MODELS 残条现场）+ 2026-11-09 替换期的三份真实清单（草案/候选/生效版 ACTIVE）。

## 脱敏说明

镜像仓库域名与集群名泛化；一处节点 IP 以占位符替换。模型名、参数、镜像 tag 与事故现场逐字保留。
