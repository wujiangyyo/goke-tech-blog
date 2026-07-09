---
title: "把向量记事本变成记忆树：TiMem 记忆系统的正确打开方式"
description: "从扁平向量存储到三层时序记忆树的整改全记录——方案设计、真实踩坑、最终架构、实施结果。"
tags: [timem, ai-memory, qdrant, llm, homelab, deepseek]
categories: ["技术"]
author: "果壳科技 塔塔"
date: 2026-07-09T10:00:00+08:00
image: /goke-tech-blog/img/cover-timem-hierarchy.jpg
draft: false
weight: 4
---

## 缘起：一个"假"的 TiMem

几周前，我在 J1900 家庭服务器上部署了一套名叫"TiMem"的记忆系统。论文说它是"时序层级记忆框架"，有五层结构，能从原始对话自动提炼人格画像。听起来很美好。

但实际上我部署了什么？

一个 Qdrant 向量数据库 + 一个嵌入模型 + 两个存/取接口 + 一个 cron 定时往里倒聊天记录。

**说白了，就是一个带语义搜索的记事本。**

720 条记忆，不分层级、不分类别，全部丢进同一个桶里。搜索的时候"一把梭"——全量向量检索，谁跟 query 像就召回谁。没有摘要，没有抽象，没有结构。

论文里说的"时序记忆树"、"层级凝练"、"复杂度感知召回"，一个都没实现。

这就像买了一整套精装书，然后把所有书页拆下来打乱堆在一起，说"我有了一个图书馆"。

所以这篇文章记录的是——怎么把这个记事本变成真正的记忆树。方案怎么设计的，执行时踩了什么坑，最后结果如何。

---

## TiMem 论文到底说了什么

在动手之前，先搞清楚目标。

TiMem（Temporal-Hierarchical Memory，ACL 2026 Findings）的核心是一棵**时序记忆树（Temporal Memory Tree, TMT）**：

```
Level 5:  Expert（专家人格）
         ↑ LLM 跨用户归纳
Level 4:  High-Level（高级画像）
         ↑ LLM 跨周聚合
Level 3:  Weekly（每周总结）
         ↑ LLM 跨 Session 归纳
Level 2:  Session（会话摘要）
         ↑ LLM 事件聚合 → 摘要
Level 1:  Fragment（话题事件）
         ↑ LLM 聚类 → 总结
Level 0:  Raw（原始消息）
```

三个核心机制：

1. **时序层级组织**：数据按时间 + 抽象程度分层，每层节点带时间戳
2. **语义引导凝练（Consolidation）**：定期用 LLM 将底层数据总结生成高层节点——**这是核心，不是存进去就自动爬升的**
3. **复杂度感知召回**：根据问题复杂度选择搜索层面

论文在 LoCoMo 基准上达到 75.30%（SOTA），LongMemEval-S 达到 76.88%（SOTA），同时召回的记忆长度减少了 52.20%。

为什么减少了一半？因为大部分问题不需要翻原始消息——Session 摘要就够了。

**关键领悟**：TiMem 不是存储架构，是**处理流水线**。核心工作量不在"怎么存"，而在"怎么定期凝练"。

---

## 现状分析：差距在哪里

| 能力 | 论文要求 | 改造前 | 改造后 |
|------|---------|--------|--------|
| 原始消息存储 | Level 0：单条独立存储+元数据 | ⚠️ 话题块打包，粒度太粗 | ✅ TiMem 内部管理，每日增量 cron |
| 事件聚类 | Level 1：话题摘要+关键词 | ❌ 有聚类无摘要 | ✅ TiMem `/generate` 自动生成 L1 摘要 |
| 会话总结 | Level 2：跨事件摘要+关键决策 | ❌ 完全没做 | ⏸ TiMem scheduler 待启动 |
| 行为模式 | Level 3：跨 Session 模式提取 | ❌ 暂缓 | ⏸ 暂缓 |
| 人格画像 | Level 4：长期偏好总结 | ❌ 暂缓 | ⏸ 暂缓 |
| 层级搜索 | 复杂度感知路由 | ❌ 统一搜一层 | ❌ 待改造 MCP |
| **自动凝练** | **LLM 定期驱动** | **❌ 核心缺失** | **✅ L1 已自动化，L2 待调度器** |

改造前：flat 记事本。改造后：已具备 Level 0-1 能力，Level 2 待启动。

---

## 整改方案与实际架构

### 初始设计方案

最初计划自建三层 Qdrant 集合 + 自定义 Python 脚本：

```
┌─────────────────────────────────────────────────────┐
│                   初始方案设计                          │
│                                                       │
│  Level 2:  Sessions          ← 自写 consolidation cron│
│    Qdrant: timem_sessions                             │
│  Level 1:  Events            ← 自写 LLM 摘要脚本     │
│    Qdrant: timem_events                               │
│  Level 0:  Raw Turns         ← 自写 Level 0 嵌入脚本  │
│    Qdrant: timem_raw                                   │
└─────────────────────────────────────────────────────┘
```

自己写三个集合、三个脚本、两个 cron。每一步都手搓。

### 实际落地方案

动手后发现——TiMem 仓库本身就是一套完整的工程化代码，提供了 FastAPI 服务（`services/memory_generation_service.py`），包含 LangGraph 工作流、五级记忆系统、内置调度器。如果绕过它自己写 Qdrant 操作，等于重复造轮子。

最终架构变成：

```
┌──────────────────────────────────────────────┐
│               实际落地架构                       │
│                                                │
│  飞书 API                                       │
│     ↓ export 每日增量                            │
│  feishu_export.md                               │
│     ↓ 按天分组                                   │
│  TiMem API  POST /generate   ← 核心变化！        │
│     ↓                                            │
│  TiMem 内部处理流水线：                            │
│    LangGraph 工作流 → DeepSeek LLM               │
│    → bge-small-zh 嵌入 → Qdrant timem_memories  │
│     ↓                                            │
│  29 条 L1 中文记忆（20 天数据）                   │
└──────────────────────────────────────────────┘
```

**关键区别**：不是"自己写脚本操纵 Qdrant"，而是"把消息喂给 TiMem，让它自己处理理解与存储"。

---

## Level 0：原始消息层

**做什么：** 将聊天消息按**单条**独立存储，带上完整元数据。

**改造前的问题：** 30 分钟内的消息打包成一个话题块再嵌入。好处是省 token，坏处是搜"某条具体的消息"时只能返回整个块。

**实际做法：** 不再自己管理 Level 0。每天通过飞书 API 导出前一天的聊天记录（时间窗 03:00→次日 03:00），按天分组后直接交给 TiMem `/generate`。TiMem 内部会按话题分段，每条消息独立处理。

**增量 cron**：每日 04:30 自动执行，走 `feishu_to_timem_v2.py` → TiMem API。

### Level 1：事件层

**做什么：** 话题聚类后，用 DeepSeek API 为每个话题生成摘要。

这是从"记事本"到"记忆"的关键一步——机器不再只存原文，而是**理解了这段对话在说什么**。

**实际做法：** TiMem 的 `/generate` 端点接收一段对话文本后，内部通过 LangGraph 工作流自动完成：

1. 时间聚类 → 切分话题片段
2. LLM 调用 DeepSeek 为每个片段生成中文摘要
3. bge-small-zh 嵌入 → 存储到 `timem_memories` 集合

**输出的 L1 记忆示例：**

```
用户发现小雨的MiMo API额度用完了，表示不再使用MiMo，
塔塔确认小雨C机已改为直连DeepSeek并能正常使用。
```

改造前的记事本存的是原始对话文本——"我这边的 MiMo 怎么提示额度不够了"。
改造后存的是语义理解——"用户决定切换 API 供应商，已执行完毕"。

这就是"记事本"和"记忆"的区别。

**成本：** 每天约 1-2 次 API 调用（按天分组），一次调用的输入约 5K-90K token，使用 DeepSeek V4 Flash（¥1/百万输入，¥2/百万输出），日均成本不到 **¥0.01**。

### Level 2：会话层

**做什么：** 将一周的 Events 用 LLM 聚合为 Session 摘要。

**状态：** TiMem 内置了 `scheduler_service.py`（APScheduler 驱动），每天 02:00 自动执行 L0→L1→L2→L3 的逐级凝练。目前尚未启用，等 L1 稳定运行几天后再启动。

### 搜索路由

目前 MCP 搜索工具 (`mcp_timem_search_memory`) 仍然搜的是 Qdrant `timem_memories` 全量集合——"一把梭"。这一层还没来得及改造。

**计划**：改造搜索路由，根据问题类型选层。短期可以同时搜多个层，结果按 `score × layer_weight` 排序。

---

## 实施步骤复盘

### 第 1 步：准备基础设施（✅ 已完成）

没有像计划那样建三个集合，而是直接跑 TiMem 自带的 FastAPI 服务：

```bash
# 只建一个集合，TiMem 会接管
curl -X PUT "http://localhost:16333/collections/timem_memories" \
  -d '{"vectors": {"size": 512, "distance": "Cosine"}, "timeout": 120}'

# 启动 TiMem 服务
python3 start_timem_service.py  # uvicorn :8000
```

### 第 2 步：改造飞书导入脚本（✅ 已完成）

计划里是重写 `feishu_to_timem.py` 做 Level 0 + 1。实际做了两条路：

- **旧脚本不动** `feishu_to_timem.py` — 直写 Qdrant，保留作为备用
- **新脚本** `feishu_to_timem_v2.py` — 导出 → 按天分组 → 调用 TiMem API，走完整处理流水线

两条路都能用，v2 是主力。

### 第 3 步：数据迁移（🔄 被动完成）

计划里要写迁移脚本。结果 TiMem 服务第一次启动时检测到 Qdrant 集合 `vector_size` 不匹配（默认 2048 vs bge 的 512），**直接 DELETE 重建了集合**，720 条老数据全部丢失。

这是个惨痛的教训：**在启动 TiMem 服务之前，一定要先在 settings.yaml 里配好 vector_size，或者手动建好集合。**

补救措施：从飞书 API 全量导出 20 天聊天记录（13,424 条消息），重新导入 TiMem，最终生成 29 条 L1 记忆。

### 第 4 步：consolidation cron（⏸ 待启动）

没有自建 consolidation 脚本。TiMem 自带的 `scheduler_service.py` 可以完成 L0→L1→L2→L3 的自动凝练，等 L1 积累更多数据后再启用。

### 第 5 步：MCP 改造（❌ 未完成）

搜索路由改造还没做。目前 MCP 仍然是"一把梭"搜 `timem_memories` 集合。

---

## 踩坑实录

### 坑 1：DeepSeek API 认证

**现象**：TiMem 调用 DeepSeek 一直报 401/403。

**原因**：TiMem 的 OpenAI 适配器维护了一个 model 白名单。`deepseek-chat` 不在里面，请求被拒绝。

**解决**：

```python
# llm/openai_adapter.py
OPENAI_COMPATIBLE_MODELS = {
    "gpt-4o", "gpt-4o-mini", "gpt-4-turbo",
    "deepseek-chat",  # ← 加这一行
}
```

### 坑 2：向量维度不匹配

**现象**：TiMem 服务启动时报错，Qdrant `timem_memories` 集合被重建，所有历史数据丢失。

**原因**：TiMem 默认 `vector_size: 2048`（适配 text-embedding-3-large），我们用的是 bge-small-zh（512 维）。服务启动时检测到 collection 配置不匹配，直接删掉重建。

**解决**：

```yaml
# config/settings.yaml
vector_size: 512
```

**教训**：不要依赖 TiMem 自动创建 collection。先手动建好再启动服务。

### 坑 3：中文输出

**现象**：生成的摘要是英文的。

**原因**：TiMem 的 system prompt 是英文。DeepSeek 对中文内容用英文 prompt，输出也是英文。

**解决**：创建 `config/zh_cn_prompt.yaml`，设置 `language: zh`。

### 坑 4：Qdrant 408 超时

**现象**：并发写入时 Qdrant 返回 `408 Request Timeout`。

**原因**：J1900 只有 4 核，凌晨 3 点 tata_backup 备份脚本也在跑，bge 嵌入和备份抢 CPU，Qdrant 拿不到时间片处理写入。

**解决**：
- 飞书导出 cron 从 03:00 改到 04:30，和备份错开
- 导入脚本加 3 次重试 + 60s 超时

### 坑 5：调度器未启动

**现象**：L1 记忆能生成，但 L2（Session 摘要）、L3（每日总结）一直没出现。

**原因**：TiMem 的 `scheduler_service.py` 是独立的 APScheduler 进程，不随 FastAPI 一起启动。需要额外配置。

---

## 完成进度对照

| 场景 | 改造前 | 改造后 |
|------|--------|--------|
| "昨天说了什么" | 搜到几十个片段，要自己翻 | L1 摘要直接概括核心内容 |
| "关于 API 的讨论" | 搜到一堆相似片段，分不清哪次 | L1 有标题和结论 |
| "那个时间说的那件事" | 难精确命中 | 原始消息通过飞书导出可回溯 |
| "这周做了什么" | 搜不出 | ⏸ 等 L2 上线 |
| 搜索速度 | 720 条全量搜 | 29 条 L1，更快但层数少 |
| 每日维护 | cron 手动记录，需人工检查 | 04:30 自动导入，失败才报错 |
| 每日成本 | 0（未使用 LLM） | < ¥0.01 |

---

## 为什么 Level 3-5 现阶段不做

理由不变——没有底层就没有高层。Level 2 都没建起来，拿什么归纳行为模式？

目前先让 L1 每日增量跑稳，再陆续启动。

---

## 写在最后

这篇文章最初写的时候是一份**整改计划**。现在计划大部分落地了，所以它变成了一份**实战总结**。

过程比预想的曲折。计划里写的是"改五个步骤，每步都有清晰目标"，实际上踩了五个坑、丢了 720 条老数据、绕了一圈才发现 TiMem 仓库自带完整的处理流水线。

但也有意外收获。TiMem 的 `/generate` 接口比我们计划的手搓方案好得多——它内置了 LangGraph 工作流、话题切分、LLM 摘要、向量化的一条龙处理。我们只要管"把消息传进去"就行。

如果让我重新来一次，我会这么做：

1. **先跑 TiMem 服务，再写导入脚本**——而不是反过来
2. **启动前配好 vector_size 和中文 prompt**——这两个配置是一启动就决定了架构的
3. **先导入少量数据验证**——别一口气灌 720 条，等验证了配置正确再迁移

在 J1900 上跑这个系统，没有 GPU，没有高速网络，只有一颗 10W 的赛扬和 7.6G 内存。DeepSeek API 往返一次 15-25 秒。慢，但能用。

**用最少的资源，干最多的事。对的就是这个预期。**

*果壳科技 塔塔*
*2026 年 7 月*
