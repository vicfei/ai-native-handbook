# ivs-llm-multi（embedding+reranker）· Pod 配置原文
Pod: ivs-llm-multi-546dc7d87f-vbljd ｜ 取证 2026-10-28（控制台终端）

## PID 1 cmdline（原文）
```
python3 /app/launcher.py --help
```
（launcher 型入口：读 MODELS 环境变量多进程拉起各 api_server）

## MODELS 环境变量（/proc/1/environ 原文全文）——⚠️ 含残条现场
```json
MODELS=[
{
  "name": "embedding",
  "command": [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--model", "/data/models/Qwen3-Embedding-4B",
    "--served-model-name", "qwen3-embedding-4b", "embedding",
    "--host", "0.0.0.0",
    "--port", "8002",
    "--enable-chunked-prefill",
    "--enable-prefix-caching",
    "--max-model-len", "8192",
    "--max-num-batched-tokens", "16384",
    "--gpu-memory-utilization", "0.45",
    "--task", "embedding"
  ]
},
{
  "name": "reranker",
  "command": [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--model", "/data/models/Qwen3-Embedding-4B",
    "--served-model-name", "qwen3-embedding-4b", "embedding",
    "--host", "0.0.0.0",
    "--port", "8002",
    "--enable-chunked-prefill",
    "--enable-prefix-caching",
    "--max-model-len", "8192",
    "--max-num-batched-tokens", "16384",
    "--gpu-memory-utilization", "0.45",
    "--task", "embedding"
  ]
},
{
  "name": "reranker",
  "command": [
    "python3", "-m", "vllm.entrypoints.openai.api_server",
    "--model", "/data/models/Qwen3-Reranker-4B",
    "--served-model-name", "qwen3-reranker-4b", "reranker",
    "--host", "0.0.0.0",
    "--port", "8003",
    "--enable-chunked-prefill",
    "--enable-prefix-caching",
    "--max-model-len", "8192",
    "--max-num-batched-tokens", "16384",
    "--gpu-memory-utilization", "0.45",
    "--task", "embedding"
  ]
}
]
```
**残条说明（10-28 三探定案）**：第二条 "reranker"（Qwen3-Embedding-4B @8002）为 embedding 条目的原样复制残条；实际仅 2 进程运行（launcher 按名去重），Embedding:8002 + Reranker:8003 双服务面 curl 正常。残条为后续编辑陷阱（改首条不生效），建议报平台组清理。

## 其他环境（原文）
```
caas_logs_stdout=true
PKS_CLUSTERNAME=<internal-cluster-name>
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
HOME=/root
```

## 运行时探针（10-28 原文）
```
ps aux|grep api_server → 1× Qwen3-Embedding-4B + 1× Qwen3-Reranker-4B（ss 不可用，镜像无该命令）
curl localhost:8002/v1/models → {"id":"qwen3-embedding-4b",...,"root":"/dat...
curl localhost:8003/v1/models → {"id":"qwen3-reranker-4b",...,"root":"/data...
```
