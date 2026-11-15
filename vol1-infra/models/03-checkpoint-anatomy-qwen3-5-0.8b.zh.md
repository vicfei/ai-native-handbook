[English](03-checkpoint-anatomy-qwen3-5-0.8b.md)

# 03 · Checkpoint 解剖：Qwen3.5-0.8B 逐张量权重普查

> 方法论一句话：**safetensors 文件头是一份 JSON 清单，读它不需要加载权重**。每个张量的参数量 = 形状逐维相乘；全文件求和即总参数量；参数量 × dtype 字节数应与文件体积吻合——账能闭合的普查才算数。

## 普查脚本（对任意 HF 模型通用，~20 行）

```python
import json, struct, math
f = open('model.safetensors-00001-of-00001.safetensors','rb')
n = struct.unpack('<Q', f.read(8))[0]          # 头部长度（前 8 字节小端）
hdr = json.loads(f.read(n)); hdr.pop('__metadata__', None)
numel = lambda k: math.prod(hdr[k]['shape'])   # 形状逐维相乘
total = sum(numel(k) for k in hdr)
print(f"{total:,} params ≈ {total/1e9:.2f}B → bf16 ×2B = {total*2/2**30:.2f} GiB")
```

## 总账（0.87B，与 1.7GB 文件互证 ✓）

| 组件 | 账单出处 | 参数量 |
|---|---|---|
| Embedding（tied 兼 LM head） | 248,320 × 1024 | 254,279,680 |
| 语言主干 24 层 | 18×21,555,360（GDN）+ 6×18,352,640（全注意力） | 498,112,320 |
| final norm | 1024 | 1,024 |
| 视觉塔（12 层 ViT + merger） | 见下文 | 100,592,896 |
| MTP 头 | 见下文 | 20,452,864 |
| **合计** | | **873,438,784 ≈ 0.87B → 1.63 GiB** |

三大件 + MTP 的结构：`Qwen3_5ForConditionalGeneration` = **ViT 视觉塔**（12 层 × 768 hidden，patch 16，merger 投影到 1024 对齐文本）+ **语言主干**（24 层，18 层 GDN 线性注意力 + 6 层全注意力按 3:1 交错：索引 3/7/11/15/19/23 为全注意力）+ **tied 词嵌入** + **MTP 旁挂头**。代码血缘（vLLM `qwen3_5.py`）：视觉前端继承 Qwen3-VL，语言骨架继承 Qwen3-Next（GDN 先驱）。

## GDN 单层账单（18 层同构）

| 张量 | 形状 | 参数量 | 角色 |
|---|---|---|---|
| in_proj_qkv | (6144, 1024) | 6,291,456 | q‖k‖v 三表合一（各 16 头×128 = 2048，×3 = 6144） |
| mlp.gate/up/down | 3×(3584, 1024) | 11,010,048 | SwiGLU 三件套 |
| in_proj_z | (2048, 1024) | 2,097,152 | 输出门 z 的原料 |
| out_proj | (1024, 2048) | 2,097,152 | 16 头×128 拼回主干宽度 1024 |
| conv1d | (6144, 1, 4) | 24,576 | depthwise 短程卷积（每通道 4 宽小窗） |
| in_proj_a / in_proj_b | 各 (16, 1024) | 32,768 | 每头一个遗忘门/写入门的输入 |
| A_log / dt_bias | (16,) | 32 | 每头衰减基线（log 参数化）与写入偏置 |
| 各 norm ×4 | — | ~2.3K | 归一化 |
| **单层合计** | | **21,555,360** | |

## 全注意力单层账单（6 层同构）

q_proj (4096,1024)=4,194,304（**含门控双份**：q 2048 + gate 2048）+ k/v_proj 各 (512,1024)=1,048,576（GQA 8Q:2KV × head_dim 256）+ o_proj (1024,2048)=2,097,152 + QK-norm 512 + 双层 norm 2,048 + MLP 11,010,048 = **18,352,640**。

门控注意力的证据链：`q_proj` 输出 = 8 头 × 256 × **(1+attn_output_gate)** = 4096，vLLM 源码 `torch.chunk(q_gate, 2)` 拆出 q 与 gate——多出的 2048 是每头输出门，不是 16 个头。

## 视觉塔与 MTP

- **视觉塔**（100.6M）：patch_embed 卷积 (768, 3, 2, 16, 16)——3 色彩通道 × 2 帧时间窗 × 16×16 像素核；12 个 ViT block（单块 7.1M：qkv 合一 + MLP + norm，**带 bias**，与语言侧无 bias 不同）；merger 把 2×2 相邻 patch 拼接（3072=768×4）压到 1024 对齐文本。
- **MTP**（20.5M）：`fc` (1024,2048) 把 [token embedding；主干末层隐状态] 两路拼接压回 1024（两路各配 pre_fc_norm）→ 一个**与主干全注意力层完全同规格**的层 → norm。复用主干 embedding/LM head，仅 20M 白得一个"预报员"（投机解码的 draft 模型）。

## 数据流全景图（GDN 层）

```
x (1024) ─┬─ in_proj_qkv(6144) → conv1d(宽4) → 拆16头×128 → q,k 过norm
          │   α = σ(in_proj_a(x) − exp(A_log))      （每头遗忘门）
          │   β = σ(in_proj_b(x) + dt_bias)          （每头写入门）
          │   递推：k̃=k−Sᵀq; S←α·S+β·v·k̃ᵀ; h=Sᵀq    （S: 每头128×128, 恒定大小）
          │   h 拼 16 头 → 2048
          ├─ in_proj_z(2048) → z
          │   输出 = out_proj(1024,2048)( SiLU(z) ⊙ h )
          └─ MLP(SwiGLU 3584) → 残差 → norm → 下一层
```

## 两个原语 + 一条形状法则（读任何解剖图的钥匙）

1. **投影 = 乘表 = 空间翻译**。一切 `_proj` 后缀都干这个：in_proj_z (2048,1024) 即"把 1024 的 x 翻译成 2048 的 z"。
2. **门 = 可学习的音量旋钮**。0~1（sigmoid）或软开关（SiLU）乘在主干信号上：z、SwiGLU 的 gate、α/β、全注意力的 q_gate 全是同一原语的不同安装位。
3. **形状法则**：投影表输入侧 = 上游宽度（残差流 1024），输出侧 = 下游需求（16 头×128=2048；三合一=6144；每头一旋钮=16）。**形状不是超参数，是被上下游夹出来的。**
