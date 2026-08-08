---
title: "MOSS-VoiceGenerator 部署实战：文字描述直接生成音色，并与 TTS-Nano 联动"
description: "用一句话描述就能凭空设计音色？MOSS-VoiceGenerator 1.7B 做到了。本文记录它在 GPU 家用机上的完整部署过程：ModelScope 拉模型、Python 3.14 + torch 2.13 环境搭建、显存 OOM 破解、Gradio 6 API 适配，以及「设计音色 → 注册 demo → 日常 TTS 合成」的联动闭环。"
tags: [AI, TTS, 语音合成, MOSS, 音色设计, 本地部署]
categories: ["技术"]
date: 2026-08-08T13:30:00+08:00
image: /goke-tech-blog/img/cover-moss-voicegenerator.jpg
draft: false
weight: 1
author: "果壳科技 塔塔"
ShowToc: true
TocOpen: true
---

# MOSS-VoiceGenerator 部署实战：文字描述直接生成音色，并与 TTS-Nano 联动

## 背景：为什么需要一个"音色设计器"

上一篇文章介绍了 MOSS-TTS-Nano——一个跑在 CPU 上的本地 TTS，支持语音克隆，注册成 demo 后日常合成语音。但它的工作流有个前置依赖：**你得先有一份参考音频**。想要"高冷御姐音""温柔睡前音"，得先找到对应的录音素材，克隆效果还取决于素材质量。

MOSS-VoiceGenerator 解决了这个问题：**输入一段文字描述，直接生成音色**（零样本、无需参考音频）。它是 MOSS-TTS 家族的姊妹模型，1.7B 参数，文字 → 音频的生成式音色设计器。

我们的目标架构：
1. **B 机**（RTX 3070Ti 8G，按需算力机）跑 VoiceGenerator，当"音色设计工作室"
2. 设计好的音色生成参考音频，拷到 **C 机**（常驻服务机）注册成 TTS-Nano 的 demo
3. 日常合成走 TTS-Nano（CPU 秒级响应），VoiceGenerator 只在设计新音色时用

```
[文字描述] → B机 VoiceGenerator (GPU 2~9s) → 参考音频 wav
                                              ↓ scp
C机 TTS-Nano (CPU 常驻) ← demo.jsonl 注册 ← wav
       ↓
  日常语音合成
```

## 一、模型与代码获取

### 1.1 模型：ModelScope 两个仓库都要下！

VoiceGenerator **不是单模型**，实际是两个：

| 仓库 | 大小 | 作用 |
|------|------|------|
| `openmoss/MOSS-VoiceGenerator` | 4.0GB | 主模型 1.7B（文字→音频 token） |
| `openmoss/MOSS-Audio-Tokenizer` | 6.7GB | 音频分词器（token↔波形编解码） |

B 机 Ubuntu 直连 ModelScope，速度 34MB/s，两个加起来 10.7GB 约 5 分钟下完。用 `resolve/master` 直链逐文件下载：

```bash
mkdir -p ~/models/MOSS-VoiceGenerator ~/models/MOSS-Audio-Tokenizer
BASE="https://modelscope.cn/models/openmoss/MOSS-VoiceGenerator/resolve/master"
for f in model.safetensors config.json tokenizer.json vocab.json merges.txt \
         added_tokens.json special_tokens_map.json processor_config.json \
         chat_template.jinja modeling_moss_tts.py configuration_moss_tts.py \
         processing_moss_tts.py inference_utils.py __init__.py; do
  curl -sL -o ~/models/MOSS-VoiceGenerator/$f "$BASE/$f"
done
# MOSS-Audio-Tokenizer 同理（model-00001/00002-of-00002.safetensors + index.json + 代码）
```

**关键点**：模型仓库自带 `modeling_moss_tts.py` 等远程代码，`trust_remote_code=True` 就能加载，**不需要 clone 整个 GitHub 仓库**。只有两个文件要从 GitHub 拿：Gradio UI 脚本 `clis/moss_voice_generator_app.py` 和示例文本 `assets/text/moss_voice_generator_example_texts.jsonl`（A 机 jsDelivr 可加速下载）。

### 1.2 代码目录结构（有坑）

Gradio 脚本里示例文本路径是写死的：

```python
EXAMPLE_TEXTS_JSONL_PATH = Path(__file__).resolve().parent.parent / "assets" / "text" / "moss_voice_generator_example_texts.jsonl"
```

所以目录必须是 `clis/app.py` + `assets/text/xxx.jsonl` 的仓库结构，**不能把 app.py 平铺在根目录**，否则启动直接 FileNotFoundError。

## 二、环境搭建（Python 3.14 + torch 2.13）

B 机系统是 Ubuntu 24.04 + Python 3.14.4。踩了两个环境坑：

### 2.1 venv 缺 ensurepip

```bash
sudo apt-get install -y python3.14-venv   # 先装 venv 包
python3 -m venv ~/mvg-venv
```

### 2.2 torch：cu121 通道没有 3.14 的 wheel！

```bash
# ❌ 失败：Could not find a version that satisfies the requirement torch
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu121
```

PyTorch 官方 cu121 索引是老通道，最高支持到 Python 3.12。**换清华默认源**，Linux 的默认 torch 自带 CUDA（实测装到 2.13.0+cu130，驱动 595.71.05 兼容）：

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip install torch torchaudio
pip install transformers gradio numpy safetensors
```

装完验证：

```bash
python -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# 2.13.0+cu130 True NVIDIA GeForce RTX 3070 Ti
```

### 2.3 缺 Python.h（triton 编译失败）

第一次跑推理时 triton 需要编译 CUDA 工具，报 `fatal error: Python.h: No such file or directory`。装开发头文件即可：

```bash
sudo apt-get install -y python3.14-dev
```

## 三、🚨 显存 OOM 破解（核心坑）

8G 显存跑两个 1.7B 模型，bf16 下各占 ~3.4GB，合计 6.8GB + 激活显存 → **必 OOM**：

```
torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 608.00 MiB.
GPU 0 has a total capacity of 7.66 GiB ... this process has 7.20 GiB memory in use
```

**解法：主模型上 GPU，Audio-Tokenizer 留 CPU**。改 app 代码一行：

```python
# 原：processor.audio_tokenizer = processor.audio_tokenizer.to(device)
processor.audio_tokenizer = processor.audio_tokenizer.to("cpu")
```

原理：processor 的 `_get_audio_tokenizer_device()` 是 best-effort 自动推断设备，tokenizer 只做音频编解码，放 CPU 慢一点点但完全可用（GPU 显存降到 ~4.3GB）。实测合成 2~9 秒/句，没有感知差异。

**另外注意**：如果 B 机 Ollama 常驻大模型（如 qwen3.6:35b 占 5.6GB），显存冲突更严重，设计音色前先 `ollama stop`。

## 四、启动与调用（Gradio 6 适配）

### 4.1 启动命令

```bash
cd ~/mvg-app
HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 \
python clis/moss_voice_generator_app.py \
  --model_path ~/models/MOSS-VoiceGenerator \
  --device cuda:0 --host 0.0.0.0 --port 7862
```

**必须带 `HF_HUB_OFFLINE=1`**！transformers 5.x 加载本地模型时也会去 HuggingFace 检查，B 机 HF 不通会卡死在 SYN-SENT（等了 2 分钟没反应才发现）。另外 `processing_moss_tts.py` 第 272 行 `codec_path` 默认指向 HF 的 `OpenMOSS-Team/MOSS-Audio-Tokenizer`，要 sed 改成 ModelScope 本地路径，否则它去 HF 找。

### 4.2 Gradio 6 API 变了

老教程都是 `POST /run/predict`，Gradio 6 已废弃。用 gradio_client：

```python
from gradio_client import Client
client = Client('http://<B机IP>:7862')
result = client.predict(
    '高冷御姐的女声，声音清冷，几乎没有情绪起伏，语速缓慢慵懒',  # instruction 音色描述
    '说完了吗？说完了就可以走了，我还有事。',                     # text 要说的话
    1.5, 0.6, 50, 1.1, 4096,                                        # 采样参数（与 Nano 不同！）
    api_name='/lambda'
)
audio_path = result[0]  # 返回 wav 文件路径
```

**采样参数与 TTS-Nano 不同**：VoiceGenerator 推荐 `temperature=1.5 / top_p=0.6 / top_k=50 / repetition_penalty=1.1`（Nano 是 0.8/0.95/25/1.2）。

### 4.3 按需启动（不常驻）

B 机是按需算力机，VoiceGenerator 低频使用，常驻会占 4.3GB 显存挡 Ollama。做成 systemd 服务 + 管理脚本：

```bash
# ~/mvg.sh start|stop|status
# start 时自动检测显存（>512MiB 提示 ollama 冲突）→ systemctl start → 等 12s
sudo systemctl stop mvg    # 用完释放显存（实际 sudo 密码勿入文）
```

## 五、联动闭环：设计音色 → 注册 demo → 日常合成

这是整套方案最有价值的部分：**VoiceGenerator 负责"创造音色"，TTS-Nano 负责"日常播报"**，一个按需重算、一个常驻轻量，各司其职。

### 5.1 设计参考音频

用上面的 API 生成一段 10~15 秒的参考音频（音色本体），例如"高冷御姐"：

```
instruction: 冷漠平淡的女声，声音清冷，几乎没有情绪起伏，语速缓慢慵懒，气息平稳不带任何热情
text: 说完了吗？说完了就可以走了，我还有事。
```

实测 4~5 秒生成 12~13 秒音频。**描述词直接影响气质**——第一次写"高冷御姐、气场强大、女总裁"生成出来像卖货主播（情绪太饱满），改成"零情绪、低能量、懒得搭理"才对味。多试几个描述词，音色方向差很多。

### 5.2 注册 C 机 demo

```bash
# 1. 参考音频拷到 C 机 voices 目录
scp mvg_ref.wav <C机>:/home/xiaoyu/projects/MOSS-TTS-Nano/voices/mvg_cold.wav

# 2. 追加 demo.jsonl（一行一条 JSON）
echo '{"name": "🎙️ MOSS高冷御姐", "role": "voices/mvg_cold.wav", "text": "说完了吗？"}' \
  >> /home/xiaoyu/projects/MOSS-TTS-Nano/assets/demo.jsonl

# 3. 重启 TTS-Nano 服务
sudo systemctl restart moss-tts.service && sleep 8

# 4. 验证新 demo_id（= 原行数+1，如 30 条后新的是 demo-31）
curl -s http://127.0.0.1:18083/ | grep 'MOSS高冷御姐'
```

**注意**：demo_id 从 1 开始编号（`len(demo_entries)+1`），别用 Python enumerate 猜，直接抓服务端 HTML 确认真实映射。

### 5.3 日常合成验证

```bash
curl -s -X POST http://127.0.0.1:18083/api/generate \
  -F 'text=找我什么事？给你一分钟，说重点。' \
  -F 'demo_id=demo-32' -o /tmp/tts.json
# 返回 JSON 里 audio_base64 是 WAV（48kHz），base64 解码即得音频
```

## 六、踩坑清单（速查）

1. **显存 OOM**：两个 1.7B 模型合计 6.8GB 超 8G → tokenizer 留 CPU（改一行）
2. **transformers 联网卡死**：加载本地模型也查 HF → `HF_HUB_OFFLINE=1`
3. **codec_path 指向 HF**：sed 改成本地 ModelScope 路径
4. **Python 3.14 + cu121 无 wheel**：用清华默认源 torch（2.13+cu130）
5. **triton 编译失败**：装 `python3.14-dev`（缺 Python.h）
6. **Gradio 6 API**：`/run/predict` 废弃，用 gradio_client + `api_name='/lambda'`
7. **示例 jsonl 路径**：目录必须 `clis/` + `assets/text/` 仓库结构
8. **pkill 自杀**：SSH 命令行含同样字符串会匹配自身进程，用 `ps aux | grep X | grep -v grep | grep -v ssh` 过滤

## 七、效果与后续

- 合成速度：**2~9 秒/句**（GPU），对比 C 机 CPU 跑 1.7B 要 1~3 分钟，量级差距
- 音色质量：文字描述能大致决定声线气质（清冷/甜糯/成熟），但**情绪细节不如真实录音克隆**——想要"完全像某个真人"还是得用参考音频克隆，VoiceGenerator 适合"凭空创造一种声音气质"
- 后续方向：SFT 微调（B 机 3070Ti 可跑，峰值 3.2GiB 显存）让音色更贴近目标；或把生成音色接入 Hermes 语音链路当默认音色

---

*果壳科技 塔塔 · 2026-08-08*
