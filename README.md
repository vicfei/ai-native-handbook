# AI-Native Handbook —— 企业 AI 岗位手册

AI-Native 企业的岗位学习手册与实践参考。三个岗位，三卷手册，同一套底座认知。

> 配套阅读：[awesome-ai-native](https://github.com/vicfei/awesome-ai-native) —— 论点驱动的 AI-Native 企业地图（转型原则、组织形态、一手文献）。手册管"怎么做"，地图管"为什么与往哪去"。

## 卷目

| 卷 | 读者 | 内容 | 状态 |
|---|---|---|---|
| [vol1-infra/](vol1-infra/) | 基础设施 / 推理优化岗 | 模型结构学习手册（checkpoint 解剖、Roofline、GDN 混合架构）；量化工具链与基准；vLLM 部署配方；kernel 学习材料 | ✅ 已有内容（自原 llm-infra-lab 并入，历史保留） |
| [vol2-agent-dev/](vol2-agent-dev/) | Agent 应用开发岗 | 企业级 RAG（开源平台二开、Java 端对接、企业部署）；企业级 Harness（部署与二开）；应用场景开发 | 🚧 骨架就绪，章节陆续迁入 |
| vol3-org/（远期） | 管理者 / 转型推动者 | 评测驱动的管理、自动化等级、组织改造操作手册 | 📋 规划中 |

## 学习路径建议

新员工从 [vol1-infra/models/](vol1-infra/models/) 的四章开始——"每 token 读全量权重"这条因果链是一切推理优化讨论的公共地基；然后按岗位分流：infra 岗走量化/评测/kernel，agent 开发岗走 vol2 的 RAG → Harness → 应用场景。

## 内容来源与发布规范

- 内容为一手实践沉淀（真实 checkpoint、真实集群、真实事故），数字可复核；
- 发布规范见 [PUBLISHING.md](PUBLISHING.md)——一句话版：技术细节完全开放，不出现公司名；凭据（key/密码/token）任何情况下不入库。

## 环境基线

vol1 实验基线：RTX 4060 Laptop 8GB (WSL2) / vLLM v0.29 / llmcompressor 0.14 / lm-eval——8GB 卡是刻意选择：装不下就逼你理解每一个显存字节去哪了。
