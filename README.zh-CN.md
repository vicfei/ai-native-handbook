<div align="center">

# AI-Native Handbook —— 企业 AI 岗位手册

**来自真实 checkpoint、真实集群、真实事故的 AI-Native 工程团队操作手册。**

English · [中文](README.zh-CN.md)

</div>

---

三卷手册、同一套底座认知、三类岗位：

| 卷 | 读者 | 内容 | 状态 |
|---|---|---|---|
| [vol1-infra/](vol1-infra/) | 基础设施 / 推理优化岗 | **模型结构学习手册**（checkpoint 解剖、Roofline 记账、GDN 混合架构）· 量化工具链与基准 · vLLM 部署配方 · kernel 学习材料 · **生产参考：一台 8×H20 上 4-pod 推理集群的逐参数档案** | ✅ 已上线 |
| [vol2-agent-dev/](vol2-agent-dev/) | Agent 应用开发岗 | 企业级 RAG（开源平台二开、Java 端对接、企业部署）· 企业级 Harness（部署与二开）· 应用场景开发 | 🚧 骨架 |
| vol3-org/（远期） | 管理者 / 转型推动者 | 评测驱动的管理、自动化等级 | 📋 规划 |

**从哪开始**：[vol1-infra/models/](vol1-infra/models/)——四章内容，从一个真实的 0.8B checkpoint 出发第一性推导"**每生成一个 token 都要完整读一遍全部权重**"，并说明这条定律如何统一解释批量、量化、MoE 与投机解码。它是手册其余一切的公共地基。

**旗舰生产内容**：[vol1-infra/production/8xh20-four-pod-reference.zh.md](vol1-infra/production/8xh20-four-pod-reference.zh.md)——4-pod 推理集群的完整配置档案（旗舰对话 / 高并发 lite / 文档解析 / 检索双件），含审计发现的六类缺陷与语义陷阱（同一个 reranker 模型，`--task embedding` 与 `--task score` 的差别就是余弦相似度与真判别概率的差别）。

> 配套地图：[awesome-ai-native](https://github.com/vicfei/awesome-ai-native) —— 论点驱动的 AI-Native 企业地图（为什么与往哪去）。本手册管"怎么做"。

## 约定

- 英文为主语言，每章配 `.zh.md` 中文版（每篇顶部有语言切换）。
- 全部一手内容：真实 checkpoint、真实集群、真实事故，数字可复现（普查脚本约 20 行 Python）。
- 发布规范见 [PUBLISHING.md](PUBLISHING.md)——技术细节完全开放（owner 裁定），公司名与凭据不出现。

## vol1 实验基线

RTX 4060 Laptop 8GB (WSL2) / vLLM v0.29 / llmcompressor 0.14 / lm-eval——8GB 卡是刻意选择：装不下就逼你为每一个显存字节记账。
