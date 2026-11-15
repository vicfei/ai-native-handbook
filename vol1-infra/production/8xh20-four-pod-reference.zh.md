# 生产参考：一台 8×H20 节点、四个推理 Pod 的完整配置档案

[English](8xh20-four-pod-reference.md)

> 离线私有云上的真实生产配置，参数与事故取证自实盘（owner 授权开放：真实参数保留，仅隐公司名/域名/凭据）。四个服务覆盖了大模型服务的四种典型形态：旗舰对话、轻量高并发、文档解析、检索双件。

## 1. 集群拓扑与约束

- 1 机 8 卡 H20（单卡 96GB HBM3），离线环境（模型/镜像全内网分发）
- 平台按 pod 分配 GPU：每 pod 可得 1/2/4/8 卡，总数 8
- 分配结果 **4+2+1+1**，且在"保四服务 + 粒度约束"下几乎是唯一解：旗舰服务（大 MoE，TP4 下权重约 50GiB/卡）物理放不进 2 卡 → 锁死 4 卡；每服务 ≥1 卡占满剩余 4 张

## 2. 四个 Pod 的完整参数

### ivs-llm-chat —— 旗舰对话（DeepSeek-V4-Flash，4 卡）

| 参数 | 值 |
|---|---|
| 引擎/镜像 | vLLM 0.25.1（vllm-docker 线） |
| 并行 | TP4 + EP4 |
| 权重 | DeepSeek-V4-Flash（FP8 出厂），NAS 存放、镜像分离 |
| 上下文/并发 | max-model-len 196608 / max-num-seqs 32 |
| KV cache | fp8，block-size 256 |
| 显存 | gpu-memory-utilization 0.9；KV offloading native 64GB |
| 投机解码 | dspark ×5（greedy） |
| 其它 | VLLM_ENGINE_READY_TIMEOUT_S=3600（大权重冷启容忍）；四别名路由（max-think/think/large 系） |

### ivs-llm-chat-lite —— 轻量高并发（Qwen3.8-27B，2 卡）

| 参数 | 值 |
|---|---|
| 并行/精度 | TP2，BF16 无量化 |
| 上下文/并发 | max-model-len 131072 / max-num-seqs 96（旗舰的 3 倍 = 高并发短请求定位） |
| KV cache | fp8_e4m3，block-size 64 |
| 显存 | gpu-memory-utilization 0.92 + PYTORCH_ALLOC_CONF expandable_segments（碎片治理） |
| 投机解码 | MTP ×2（checkpoint 自带头） |
| 混合架构配套 | --mamba-cache-mode align（GDN 线性注意力层的状态缓存模式） |
| 工具调用 | qwen3_coder parser + auto-tool-choice |
| 探针 | 仅 liveness（initialDelay 600s）——审计缺陷 #1 |

### ivs-llm-multi —— 检索双件（Embedding+Reranker，1 卡）

单容器双实例形态：launcher 进程读 `MODELS` 环境变量拉起两个 vLLM（vllm-dynamic 0.10.1 线）。修复后的正确配置：

```yaml
- name: MODELS
  value: |
    [
      { "name": "embedding",
        "command": ["python3","-m","vllm.entrypoints.openai.api_server",
          "--model","/data/models/Qwen3-Embedding-4B",
          "--served-model-name","qwen3-embedding-4b","embedding",
          "--host","0.0.0.0","--port","8002",
          "--max_model_len","8192","--max_num_batched_tokens","16384",
          "--gpu-memory-utilization","0.40",
          "--task","embedding"] },
      { "name": "reranker",
        "command": ["python3","-m","vllm.entrypoints.openai.api_server",
          "--model","/data/models/Qwen3-Reranker-4B",
          "--served-model-name","qwen3-reranker-4b","reranker",
          "--host","0.0.0.0","--port","8003",
          "--max_model_len","8192","--max_num_batched_tokens","8192",
          "--gpu-memory-utilization","0.28","--enforce-eager",
          "--task","score",
          "--hf_overrides","{\"architectures\":[\"Qwen3ForSequenceClassification\"],\"classifier_from_token\":[\"no\",\"yes\"],\"is_original_qwen3_reranker\":true}"] }
    ]
```

三个关键语义（踩过坑换来的）：

1. **reranker 必须是 `--task score` + hf_overrides 三件套**：从 LM head 取 no/yes 两行做差构造单行判别头（数学上等价于官方 yes/no softmax），`/v1/rerank` 返回真判别概率；用 `--task embedding` 起（旧配置）分数会退化为"未按对比训练的隐向量做余弦"——接口通、排序看起来也通、语义错；
2. **embedding 实例的 `--task embedding` 一个字不能丢**：丢了 → 引擎按架构默认走 generate → `/v1/embeddings` 404 → 上游入库/召回全断，且是静默断（端口照样监听、健康口照样 200）；
3. util 0.40/0.28 + batch 8192 + enforce-eager 是 OOM 修复带：rerank 大请求的激活峰值曾在 0.45+0.45+batch 16384 下击穿余量（运行期死亡 10 小时无人察觉的事故）。

### ivs-llm-file-parser —— 文档解析（1 卡）

- 双容器：解析主容器（视觉管线，4.1.3 线，**镜像与权重打包**——与其余三 pod 的 NAS 分离路线相反，每次升级需重建数十 GB 镜像）+ 格式转换容器
- 显存语义：tier 自动探测（无显式 mem-fraction）；API 从同步 /file_parse 单端点换代为异步任务制（V1 契约不兼容，调用方需适配）

## 3. 审计发现的六类缺陷（配置治理参考）

| # | 问题 | 根因 | 修正方向 |
|---|---|---|---|
| 1 | 仅 liveness，无 readiness/startup | 用 initialDelay 常数猜冷启时长 → 滚动更新流量黑洞 + 慢载死循环 | 探针三件套 |
| 2 | util 0.90~0.92 贴顶无压测 | 决策无证据链；激活峰值可击穿 | 压测阶梯定档 + 显式 max-num-batched-tokens |
| 3 | MTP×2 与 96 槽位同配 | 两个参数描述相反的服务形态 | 接受率×并发曲线分档 / adaptive verification |
| 4 | 96×128K 容量口径失真 | 名义积远超 KV 池 | 按"并发×平均上下文"重算 + 排队上限 |
| 5 | 检索任务语义错配 | task 旗标语义分叉，接口通≠语义对 | 语义验收进上线清单 |
| 6 | 引擎版本-架构代际错配 | 新架构跑旧引擎，修复未吃到 | 升级回归清单化 |

共同根因：**配置交付没有证据链**。修每一项时把压测曲线/语义验收/取证记录一并落账，是从"模板交付"走向"可审计运维"的最短路径。
