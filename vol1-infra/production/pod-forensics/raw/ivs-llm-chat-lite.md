# ivs-llm-chat-lite · Pod 配置原文
Pod: ivs-llm-chat-lite-865997c6c9-g9td4 ｜ 取证 2026-10-28（控制台终端）

## PID 1 cmdline（原文）
```
python3 -m vllm.entrypoints.openai.api_server --model /data/models/Qwen3.8-27B --served-model-name qwen3.8-27b lite --trust-remote-code --dtype bfloat16 --tensor-parallel-size 2 --max-model-len 131072 --kv-cache-dtype fp8_e4m3 --gpu-memory-utilization 0.92 --block-size 64 --enable-auto-tool-choice --tool-call-parser qwen3_coder --reasoning-parser qwen3 --max-num-seqs 96 --max-num-batched-tokens 16384 --enable-chunked-prefill --enable-prefix-caching --mamba-cache-mode align --speculative-config '{"method":"mtp","num_speculative_tokens":2}' --disable-custom-all-reduce --host 0.0.0.0 --port 8000
```

## 环境（/proc/1/environ 原文，grep vllm|model 段）
```
VLLM_ENABLE_CUDA_COMPATIBILITY=0
VLLM_USAGE_SOURCE=production-docker-image
VLLM_BUILD_COMMIT=752a3a504485790a2e8491cacbb35c137339ad34
VLLM_BUILD_PIPELINE=019d130e-464e-4ff7-b84b-492992c0c06b
VLLM_BUILD_URL=https://buildkite.com/vllm/release-v2/builds/3803
```
（与 ivs-llm-chat 引擎 build 逐字一致）

## 模型 config.json 提取
```
arch: ['Qwen3_5ForConditionalGeneration']   # 与 Qwen3.5-0.8B 同架构族
name_or_path: （空）
quant: （空 = BF16 无量化）
```

## 外部观察（办公机，2026-10-28）
- GET /version → {"version":"0.25.1"}
- GET /v1/models → qwen3.8-27b / lite
