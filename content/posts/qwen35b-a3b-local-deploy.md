---
title: "RTX 3070 Ti 本地部署 Qwen3.6-35B-A3B：8G 显存 + 16G 内存跑 35B MoE 多模态全流程"
description: "从避坑教程识别到 GGUF 量化选型，从 Ollama 版本排障到视觉实测，完整记录在 RTX 3070 Ti 上跑通 Qwen3.6-35B-A3B（35B 总参/3B 激活 MoE）的实战经验。"
tags: [ollama, qwen, moe, gguf, multimodal, local-llm, homelab, rtx3070ti]
categories: ["技术"]
author: "果壳科技 塔塔"
date: 2026-08-03T08:00:00+08:00
draft: false
weight: 4
image: /goke-tech-blog/img/cover-qwen35b-a3b.jpg
ShowToc: true
TocOpen: true
---

## 背景

我一直想把本地 GPU 用起来跑大模型。B 机（RTX 3070 Ti 8G + 16G 内存 + Ubuntu）是现成的算力节点，但"8G 显存能跑什么"这个问题，答案远比网上教程复杂。

事情的起因是有人分享了一份"Ubuntu + RTX 3070 Ti 部署 Qwen3.6-35B-MoE"的教程，号称 20-25 tokens/s、显存 7.5G、支持多模态。听着很诱人，但验证下来教程**半真半假**——模型是真的，命令是编的。

这篇文章记录真实的部署路径：**Ollama + GGUF 低量化 + 实测数据**，不吹不黑。

---

## 模型选型：35B MoE 是怎么回事

Qwen3.6-35B-A3B 是阿里开源的稀疏混合专家（MoE）模型：

- **总参数 35B**，但每次推理**只激活 3B 参数**（A3B = Active 3B）
- 原生**多模态**：内置视觉编码器，能看图（vision）
- 还支持 tools（工具调用）和 thinking（思维链）
- 对比同代 27B 稠密模型，MoE 在推理速度上有代差优势

**为什么 MoE 适合小显存？** 因为推理时只需要把"被激活的专家"加载到显存，未激活部分留在内存。这让 8G 显存 + 16G 内存的组合有了跑 35B 的可能性——但前提是量化得当。

---

## 踩坑一：别信"教程"，先查证

那份教程声称：
- ❌ ModelScope 仓库页不存在（下载命令是编的）
- ❌ hanshuixin.org 预编译包 403（可疑站点）
- ❌ 手动装 CUDA 12.4 多余（驱动自带 CUDA 13.2）
- ❌ `--mmap` 解释错误

**正确姿势**：模型是真实存在的，`qwen3.6:35b-a3b` 就在 Ollama 官方库和 ModelScope 的 unsloth GGUF 仓库里。走正规渠道，别碰来路不明的预编译包。

---

## 量化选型：24GB 必 swap，13GB 刚好

Ollama 官方默认 tag 是 **Q4_K_M 24GB**——这个体积对 8G 显存 + 16G 内存来说**必严重 swap**（显存 8G + 内存 16G = 22G < 24G）。

所以要从 unsloth 的 GGUF 仓库选低量化版本：

| 量化 | 体积 | 结论 |
|:----:|:----:|:----:|
| Q4_K_M | 22.1 GB | ❌ 必 swap |
| Q3_K_M | 16.6 GB | 🟡 贴边，有风险 |
| **IQ3_S** | **13.7 GB** | ✅ 8G 显存 + 剩余内存刚好装下 |
| IQ3_XXS | 13.2 GB | ✅ 更小，质量略降 |

**我选了 IQ3_S（3.44 bpw）**——质量和体积的平衡点。视觉投影模块 `mmproj-F16.gguf`（0.9GB）也要一起下载，视觉能力全靠它。

---

## 踩坑二：ModelScope 下载被限速

ModelScope 直链下载踩了个隐蔽的坑：

```bash
# ❌ 加了 range 头，被 CDN 限速到 1KB/s
curl -r 0-5000000 URL

# ✅ 直接跟重定向下载，34MB/s
curl -sL -o model.gguf URL
```

**加了 `-r`（range）会被 CDN 当成断点续传限速**，去掉后满速。13.7GB 大约 7 分钟拉完。

---

## 踩坑三：Ollama 版本太旧，不认识新架构

GGUF 导入 Ollama 后加载报错：

```
llama_model_load: error loading model architecture: unknown model architecture: 'qwen35moe'
```

**根因**：Ollama stable 还停在 v0.24.0（2026-05），内置 llama.cpp 不认识 Qwen3.6 的新架构 `qwen35moe`。升级到 **edge 通道 v0.30.10**（2026-07）解决：

```bash
sudo snap refresh ollama --edge
```

---

## 部署命令汇总

```bash
# 1. 下载 GGUF（unsloth 官方仓库，走 ModelScope）
mkdir -p ~/models/qwen36 && cd ~/models/qwen36
curl -sL -o Qwen3.6-35B-A3B-UD-IQ3_S.gguf \
  "https://modelscope.cn/models/unsloth/Qwen3.6-35B-A3B-GGUF/resolve/master/Qwen3.6-35B-A3B-UD-IQ3_S.gguf"
curl -sL -o mmproj-F16.gguf \
  "https://modelscope.cn/models/unsloth/Qwen3.6-35B-A3B-GGUF/resolve/master/mmproj-F16.gguf"

# 2. 创建 Modelfile（FROM 主模型 + ADAPTER 视觉投影）
cat > Modelfile << 'EOF'
FROM ./Qwen3.6-35B-A3B-UD-IQ3_S.gguf
ADAPTER ./mmproj-F16.gguf
TEMPLATE """{{- if .System }}<|im_start|>system
{{ .System }}<|im_end|>
{{- end }}<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
PARAMETER stop "<|im_start|>"
PARAMETER stop "<|im_end|>"
EOF

# 3. 导入 Ollama
ollama create qwen3.6:35b-a3b-iq3s -f Modelfile

# 4. 实测
ollama run qwen3.6:35b-a3b-iq3s
```

---

## 实测数据（真实跑出来的）

### 文本推理

| 场景 | 结果 |
|:----:|:----:|
| 首次加载耗时 | ~130s（14GB 读入） |
| 热加载耗时 | 0.3s |
| 文本生成速度 | **13.9 tokens/s** |
| 显存占用 | 5.6GB / 8GB |
| 内存占用 | 无 swap，12Gi available |

### 视觉推理

| 场景 | 结果 |
|:----:|:----:|
| 测试图识别（程序生成） | ✅ 正确识别"日本国旗"（红圆+白底） |
| 真实照片识别 | ✅ 商场场景、人物、中文小字全部识别 |
| 视觉推理速度 | **32.4 tokens/s** |

**最惊喜的是真实照片测试**：照片里的商场、小女孩、甚至背景墙上的中文标语「来万达一起浪吧」「万趣留言墙」全部准确读出——中文小字 OCR 级别的识别能力。

---

## 结论与展望

1. **8G 显存 + 16G 内存跑 35B MoE 完全可行**，前提是选对量化（IQ3_S）和模型架构（MoE 激活参数少）
2. **视觉能力是原生内置的**，不需要额外配置，加 `images` 字段即可
3. 13.9 tokens/s 的文本速度足够日常对话和编码辅助；视觉 32 tokens/s 甚至更快
4. 这套组合的下一步：给小米摄像头"当眼睛"——取流 + 视觉模型实时分析

**避坑总结**：教程先验证再执行（模型真实性、仓库是否存在）、量化版本看体积选（Q4 必 swap 就别硬上）、工具链版本要对齐（Ollama edge 才认识新架构）、下载别加 range 头（被 CDN 限速）。
