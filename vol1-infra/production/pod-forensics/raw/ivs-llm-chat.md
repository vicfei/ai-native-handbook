# ivs-llm-chat（flash）· Pod 配置原文
Pod: ivs-llm-chat-59c7b69bcf-617dn ｜ 取证 2026-10-28（控制台终端）

## PID 1 cmdline（原文）
```
python3 -m vllm.entrypoints.openai.api_server --model /data/models/DeepSeek-V4-Flash-0731 --served-model-name deepseek-v4-flash-dspark large-max-think large-think large --trust-remote-code --dtype auto --tensor-parallel-size 4 --enable-expert-parallel --max-model-len 196608 --kv-cache-dtype fp8 --block-size 256 --gpu-memory-utilization 0.9 --tokenizer-mode deepseek_v4 --kv-offloading-backend native --kv-offloading-size 64 --enable-auto-tool-choice --tool-call-parser deepseek_v4 --reasoning-parser deepseek_v4 --speculative-config '{"method":"dspark","num_speculative_tokens":5,"draft_sample_method":"greedy"}' --max-num-seqs 32 --max-num-batched-tokens 16384 --enable-chunked-prefill --enable-prefix-caching --host 0.0.0.0 --port 8000
```

## 环境（/proc/1/environ 原文，grep vllm|model 段）
```
VLLM_ENABLE_CUDA_COMPATIBILITY=0
VLLM_USAGE_SOURCE=production-docker-image
VLLM_BUILD_COMMIT=752a3a504485790a2e8491cacbb35c137339ad34
VLLM_BUILD_PIPELINE=019d130e-464e-4ff7-b84b-492992c0c06b
VLLM_BUILD_URL=https://buildkite.com/vllm/release-v2/builds/3803
```

## 模型 config.json 提取
```
arch: ['DeepseekV4ForCausalLM']
quant: fp8
```

## 外部观察（办公机，2026-10-28）
- GET /version → {"version":"0.25.1"}
- GET /v1/models → deepseek-v4-flash-dspark / large-max-think / large-think / large（python 解析 JSON 复核，四别名）
