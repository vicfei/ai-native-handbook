# ivs-llm-file-parser（MinerU）· Pod 配置原文
Pod: ivs-llm-file-parser-7cb98fdd4c-2zwms ｜ 取证 2026-10-28 ｜ 三容器：mineru-api / sglang-server / memect-parser

## mineru-api 容器
PID 1 cmdline（原文）：
```
/usr/bin/python3 /usr/local/bin/mineru-api --host 0.0.0.0 --port 8000 --enable_sglang_engine true
```
版本（pip 原文）：
```
Name: mineru
Version: 2.2.2
```
日志形态：uvicorn 无 --no-access-log 旗标（access log 默认开，stdout）；容器内无 .log 落盘文件（find 仅 /var/log 系统日志）；/usr/local/bin/mineru-models-download 在位；grep 可用。

## API 面（办公机探针原文）
```
GET /openapi.json → info: {'title': 'FastAPI', 'version': '0.1.0'}；paths: 仅 /file_parse
GET /version → {"detail":"Not Found"}
```
参数（openapi 权威）：files / output_dir / lang_list / backend 四参数。

## 实弹测试记录（办公机 curl，2026-10-28 原文）
```
backend=pipeline        → 200，22.8s，{"backend":"pipeline","version":"2.2.2","results":{"test":{"md_content":"...（部门专项学习基金使用指南）..."
backend=vlm             → {"error":"Failed to process file: Unsupported backend: vlm. Supports: transformers, sglang-engine, sglang-client."}
backend=sglang-engine   → {"error":"Failed to process file: sglang is not installed, so sglang-engine backend cannot be used. If you need to use sglang-engine backend for inference, please install sglang[all]==0.4.8 or a newer version."}
```
（backend=sglang-client 10-27 曾实测报 server_url must be provided；backend=transformers 未测）

## sglang-server 容器
- workspace：/sgl-workspace/sglang（sglang 源码树，2025-08-03 时间戳）
- 镜像极简（无 grep）
- **运行状态未验证**（VLM 档三层缺口之一：mineru-api 缺 sglang 包 / API 无 server_url 通道 / 本容器是否载有 VLM 权重未探）

## memect-parser 容器
- 入口未探（PID 1 cmdline 未取证）——平台定制网关层（四参数裁剪 API 的出处）
