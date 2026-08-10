---
title: "给 Hermes 装一套语音助手：旧手机当耳朵、小爱当嘴巴、大模型当大脑"
date: 2026-08-10
draft: false
tags: ["Hermes", "语音助手", "MOSS-TTS", "DLNA", "视觉识别", "唤醒词"]
categories: ["AI Agent"]
author: "果壳科技 聂小雨"
summary: "把旧手机变成耳朵和眼睛、小爱音箱变成嘴巴，加上 Hermes 0.20.0 做大脑，搭了一个隔空喊话的智能音箱式语音助手。记录完整链路、实现方式和踩过的坑。"
image: "/goke-tech-blog/img/cover-xinnian-voice.jpg"
ShowToc: true
TocOpen: true
---

# 给 Hermes 装一套语音助手

喊一声"新年，现在几点"，家里的旧手机收音、唤醒、转写、大模型回答、语音合成、小爱音箱播报，全自动走完。这篇文章记录这条链路的完整实现：架构、每个环节的选型与坑、以及关键源码 patch。

## 架构总览

```
旧手机 (android-ip-camera)
 ├─ 耳朵: /audio/raw 原始麦克风流 → systemd 服务 → PipeWire 虚拟麦克风
 ├─ 眼睛: /video/snapshot 抓帧 → 缩小到 512 → qwen3-vl 视觉分析
 └─ 嘴巴: 小爱音箱 DLNA 播报

服务器 (Hermes 0.20.0):
 ├─ phone-ear.service     ffmpeg 拉流灌虚拟麦克风（开机自启）
 ├─ moss-tts.service      MOSS-TTS-Nano 语音合成
 ├─ nginx                DLNA 音频静态服务
 ├─ Hermes CLI (新年 profile)  唤醒 + STT + 大脑 + TTS
 └─ Ollama (局域网另一台机器)   qwen3-vl:8b 视觉模型
```

完整调用链：手机收音 → 虚拟麦克风 → sherpa 中文唤醒 → faster-whisper 转写 → deepseek 生成回答（qwen3-vl 视觉辅助）→ MOSS-TTS 合成 → 推流 → 小爱 DLNA 播报。

## 耳朵：手机收音进虚拟麦克风

旧手机装 android-ip-camera（开源，F-Droid 可装），HTTP 服务暴露麦克风和摄像头端点。音频端点有两个：`/audio`（处理流，带降噪，实测曾全静音）和 `/audio/raw`（原始麦克风，绕过手机降噪/回声消除/AGC，更干净，最终采用）。

服务器侧一个 systemd 服务（phone-ear.service）跑 ffmpeg：带 Basic 认证拉 `/audio/raw` → 增益 → 转 48kHz 双声道 → 灌进 PipeWire 虚拟麦克风。Hermes 不需要知道手机存在，只读本地虚拟麦，整个语音栈（唤醒/STT）直接复用。

## 唤醒：中文唤醒词"新年"

Hermes 内置的 sherpa 唤醒引擎默认只支持英文——gigaspeech 模型全是英文 BPE token，中文短语直接报 "Can't find token"。解决办法是 patch 引擎支持拼音分词 + 换中文模型：

- `tools/wake_word.py` 从配置读 `tokens_type`（默认 "bpe"），中文走 ppinyin 分词
- 模型换成 sherpa-onnx 的 wenetspeech（拼音 token，中文友好）
- 唤醒词设"新年"两个字——短词触发更早，喊"给新年"也包含"新年"能触发

⚠️ 源码 patch 会被 `hermes update` 覆盖，升级后需重打。

## STT：本地语音转写

faster-whisper small 模型本地转写中文。两个坑：模型必须用本地缓存的版本（配 base 会联网下载卡死）；CLI 会话要设 `HF_HUB_OFFLINE=1` 防 huggingface 联网检查卡住。

## 大脑：完整 Hermes 核心

语音助手跑在 Hermes 新年 profile 上：主模型 deepseek，视觉辅助 qwen3-vl:8b（局域网另一台机器的 Ollama），人格、记忆、技能、工具全在框架内。这是一条架构红线：**语音助手必须依赖 Hermes 完整核心，不写独立脚本绕开**——否则等于把大脑扔了，只剩一个玩具。

## 嘴巴：MOSS-TTS + 小爱 DLNA

TTS 用 MOSS-TTS-Nano（本地部署，纯 CPU 推理）。音色对比了一圈：温柔晚安声音太低、高冷御姐像老太太、邻家姐姐怪，最后选了杨幂音色，最自然；自己克隆的音色远不如官方调教的。

Hermes 的 TTS command provider 契约要求"返回音频文件"，所以 wrapper 走双路：合成后正常返回音频给 Hermes，另转 mp3 推给小爱。mp3 前加 2 秒静音吃小爱启动渐入、后加 2 秒静音做 Stop 窗口。UPnP 推流（SetAVTransportURI + Play）后固定延时发 Stop，正好掐干净。

## 眼睛：视觉识别

手机 `/video/snapshot` 抓帧（4000×3000 级别原图），**必须缩小到 512 再喂视觉模型**。qwen3-vl:8b 实测基准：512 图约 45 秒、1024 图 3-4 分钟、4K 原图跑不动。新年自己抓帧时会读 phone-av-sensor 技能自动缩小，实测能自主完成"抓帧→缩小→识别"，准确描述出画面内容。

## 关键配置表

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 唤醒模型 | sherpa-onnx wenetspeech 3.3M | 中文拼音 token |
| 唤醒词 | 新年 | 短词触发更早 |
| STT 模型 | faster-whisper small | 本地缓存 |
| VAD 阈值 | 80 | 必须明显高于底噪 |
| VAD 静音时长 | 2.0s | 说完自动停 |
| 虚拟麦增益 | 4x | 底噪/说话分离度 |
| TTS 音色 | demo-6 杨幂 | 最终选定 |
| mp3 前后静音 | 前 2s + 后 2s | Stop 窗口与防重播 |

## 重要踩坑

1. **唤醒引擎不支持中文**：gigaspeech 英文模型零中文 token；sherpa 原生能跑 wenetspeech，但 Hermes 硬编码 BPE 分词，patch 引擎支持 ppinyin 解决。
2. **VAD 阈值必须实测**：阈值设 50 恰好压在底噪上，VAD 永远判不出静音，录音不停。实测底噪后定 80。
3. **Stop 时机三坑**：按推流时刻估算会被拉流延迟掐断；等整段播完会把自动重播开头漏出来（听感"播了一遍多一点"）；轮询播放状态不可靠——小爱响应慢时轮询全超时，mp3 循环播多遍。最终固定延时 `前导+语音+0.3s` 最稳。
4. **"No speech detected" 双坑**：录音自动停用配置阈值，但录音后"太轻丢弃"判定是硬编码——要 patch 成统一用配置值，否则低音量远场音频白录。
5. **唤醒触发但没声音**：CLI 的 TTS 默认关闭，唤醒流程要显式开启。
6. **ffmpeg 偶发卡死**：进程在、CPU 时间极低、虚拟麦静音 → 重启服务即可，建议加 watchdog。
7. **STT 听不清（未完全解决）**：手机远场收音 + 环境声干扰，转写不稳定，排查方向是物理解决信噪比。

## 源码 patch 清单

1. `tools/wake_word.py`：sherpa 引擎支持 tokens_type 动态传参（ppinyin 中文分词）
2. `cli.py`：唤醒流程同步开启 TTS 播报
3. `tools/voice_mode.py`："太轻丢弃"判定改用配置阈值

升级 Hermes 后需重打以上 patch。

## 待办

- 生产形态 systemd 服务化（不用管终端，开机自启）
- ffmpeg watchdog 自动重启
- STT 转写质量的进一步优化
