---
title: "把向量记事本变成记忆树：TiMem 记忆系统的正确打开方式"
description: "我们从 TiMem 论文出发，审视了当前简化实现的不足，提出了从扁平向量存储升级为三层时序记忆树的具体整改计划。"
tags: [timem, ai-memory, qdrant, llm, homelab]
categories: 技术
date: 2026-07-08T23:50:00+08:00
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

所以这篇文章是一份**整改计划**——怎么从记事本变成真正的记忆树。

---

## TiMem 论文到底说了什么

在动手之前，先搞清楚目标。

TiMem（Temporal-Hierarchical Memory，ACL 2026 Findings）的核心是一棵**时序记忆树（Temporal Memory Tree, TMT）**：

```
Level 4:  Persona（人格画像）
         ↑ LLM 跨 Session 归纳
Level 3:  Pattern（行为模式）
         ↑ LLM 跨 Session 总结
Level 2:  Session（会话摘要）
         ↑ LLM 事件聚合 → 摘要
Level 1:  Event（话题事件）
         ↑ 时间聚类 → 话题切块
Level 0:  Raw Turn（原始消息）
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

| 能力 | 论文要求 | 我们现状 |
|------|---------|---------|
| 原始消息存储 | Level 0：单条独立存储+元数据 | ⚠️ 有，但打包成话题块存，粒度太粗 |
| 事件聚类 | Level 1：话题摘要+关键词 | ❌ 有话题聚类但无 LLM 摘要 |
| 会话总结 | Level 2：跨事件摘要+关键决策 | ❌ 完全没做 |
| 行为模式 | Level 3：跨 Session 模式提取 | ❌ 暂缓 |
| 人格画像 | Level 4：长期偏好总结 | ❌ 暂缓 |
| 层级搜索 | 复杂度感知路由 | ❌ 统一搜一层 |
| **自动凝练** | **LLM 定期驱动** | **❌ 核心缺失** |

---

## 整改目标：实现 Level 0 → 1 → 2

### 总体架构

```
┌─────────────────────────────────────────────────────┐
│                   TiMem Memory Tree                  │
│                                                       │
│  Level 2:  Sessions          ← LLM 聚合（每周 cron）  │
│    Qdrant: timem_sessions                             │
│    内容: 会话摘要、关键决策、时间范围                   │
│           ↑                                            │
│  Level 1:  Events            ← LLM 摘要（每日 cron）   │
│    Qdrant: timem_events                                │
│    内容: 话题标题、摘要、关键词、关联消息数              │
│           ↑                                            │
│  Level 0:  Raw Turns         ← 消息摄入（准实时）       │
│    Qdrant: timem_raw                                   │
│    内容: 单条消息、用户、时间戳、来源                   │
│                                                       │
│  搜索层: 分析问题 → 选层 → 检索 → 融合返回             │
└─────────────────────────────────────────────────────┘
```

### Level 0：原始消息层

**做什么：** 将聊天消息按**单条**独立存储，带上完整元数据。

**当前做法的问题：** 把 30 分钟内的消息打包成一个话题块再嵌入。好处是省 token（1条嵌入 vs N条嵌入），坏处是丢失了单条消息的搜索精度——搜"某个具体时间说的那句话"时，只能返回整个话题块。

**整改方案：**

```python
# Level 0 单条消息结构
{
    "id": "uuid",
    "text": "原始消息文本",
    "user_id": "ou_xxx",
    "user_name": "你/塔塔",
    "timestamp": "2026-07-08T10:30:00+08:00",
    "source": "feishu_daily",
    "topic_id": "关联的 Event ID",
    "vector": [0.1, 0.2, ...]  # bge-small-zh 嵌入
}
```

**改动量：** 修改 `feishu_to_timem.py`——每条消息单独嵌入+存储，同时带上 `topic_id` 预关联到话题分组。

### Level 1：事件层

**做什么：** 话题聚类后，**用 DeepSeek API 为每个话题生成摘要**。

这是从"记事本"到"记忆"的关键一步——机器不再只存原文，而是**理解了这段对话在说什么**。

**Event 结构：**

```python
{
    "id": "uuid",
    "title": "讨论小雨 API 配置变更",     # LLM 生成
    "summary": "用户决定将小雨的 API 提供者从 xiaomi(MiMo) 切换为 deepseek 直连，节省约 87% 费用。",  # LLM 生成
    "messages_count": 12,
    "participants": ["你", "塔塔"],
    "time_range": {"start": "...", "end": "..."},
    "tags": ["配置变更", "API", "小雨"],  # LLM 提取
    "raw_ids": ["level0_id1", ...],
    "vector": [0.1, 0.2, ...]
}
```

**LLM Prompt：**

```
你是一个记忆摘要助手。以下是同一话题的一组对话消息，
请生成：
1. 标题（10字以内）
2. 摘要（3句话以内，概括核心内容和结论）
3. 标签（3-5个关键词）

消息：
[消息1]
[消息2]
...
```

**为什么必须用 LLM：**
- 话题切块只解决了"哪些消息属于同一段对话"，但不知道"这段对话在说什么"
- LLM 可以从对话中提取**结论、决策、关键词**——这是语义理解，不是向量相似度能替代的
- 后续搜索时，Event 摘要比原始消息更浓缩、更准确

**成本评估：** 每天约 10-20 个话题块，每次 LLM 调用消耗约 500-2000 token（取决于话题长度）。按 DeepSeek V4 Flash ¥1/百万输入、¥2/百万输出的价格，日均成本不到 **¥0.01**。可以忽略不计。

### Level 2：会话层

**做什么：** 每周一次，将过去一周的 Events **用 LLM 聚合**为 Session 摘要。

**Session 结构：**

```python
{
    "id": "uuid",
    "title": "7月第二周工作汇总",
    "summary": "本周主要完成了 TiMem 整改规划、小雨 API 切换、J1900 技术笔记撰写",
    "key_decisions": [
        "小雨切换 deepseek 直连",
        "TiMem 增加 Level 0-2 层级",
        "发布 J1900 家庭服务器技术笔记"
    ],
    "key_topics": ["API优化", "记忆系统", "技术写作"],
    "event_ids": ["event_1", "event_2", ...],
    "date_range": {"start": "2026-07-06", "end": "2026-07-12"},
    "vector": [0.1, 0.2, ...]
}
```

**凝练时机：** 每天的飞书导出 cron 跑完后触发（如果当日有新增 Events）。或者每周日汇总一次。

### 搜索路由

搜索时不再"一把梭"，而是根据问题类型选层：

| 问题类型 | 示例 | 搜索层 | 理由 |
|---------|------|-------|------|
| 具体事实 | "我昨天说了什么" | Level 0 | 需要精确命中某条消息 |
| 话题回顾 | "关于 API 的讨论" | Level 1 | Event 摘要更浓缩 |
| 周/月总结 | "这周做了什么" | Level 2 | Session 天然按周组织 |

**最低成本实现：** 同时搜三个层级，结果按 `score × layer_weight` 排序返回。Layer 权重可以调（如 Level 1 权重更高，因为它最常被需要）。这样不需要预先判断问题类型——系统自动"哪个层级的结果更相关就用哪个"。

---

## 实施步骤

### 第 1 步：准备 Qdrant 集合

```bash
# 新建 3 个集合
for col in timem_raw timem_events timem_sessions; do
  curl -X PUT "http://localhost:16333/collections/$col" \
    -H "Content-Type: application/json" \
    -d '{"vectors": {"size": 512, "distance": "Cosine"}}'
done

# 旧集合 timem_memories 保留暂不删除
```

### 第 2 步：改造飞书导出脚本

重写 `feishu_to_timem.py`：
- 每条消息独立嵌入 → Level 0 ✅
- 话题聚类后调 LLM 生成摘要 → Level 1 ✅
- 保留现有话题切块逻辑（30 分钟间隔）

### 第 3 步：新增 consolidation cron

新建 `/data/塔塔工作站/scripts/timem_consolidate.py`：
- 每天/每周扫描新增 Events
- 用 DeepSeek API 生成 Session 摘要
- 存入 Level 2

### 第 4 步：改造 MCP 工具

改造 `timem_mcp.py`：
- `store_memory` → 支持指定层级
- `search_memory` → 支持跨层级搜索 + 路由
- 保留向后兼容（默认搜全量）

### 第 5 步：老数据迁移

现有的 720 条 `timem_memories` 数据：
- 616 条历史会话导出 → 批量调 LLM 生成 Events → Level 1
- 104 条飞书导出 → 拆回单条消息 → Level 0
- 迁移完成后再考虑下架旧集合

---

## 为什么 Level 3 和 Level 4 现阶段不做

### Level 3（行为模式）

**需要什么：** 至少数十个 Session 数据，跨 Session 的模式识别。

**为什么不做：** Level 2 都还没建起来，没有素材可以归纳。至少运行 1-2 个月，积累 30+ 个 Session 才有意义。

### Level 4（人格画像）

**需要什么：**
- Level 3 的 Pattern 作为输入
- 跨长时间维度的人格提炼
- 人工验证准确性

**为什么不做：**
- 人格归纳需要跨长时间维度的模式数据
- 人格信息目前由 Hermes 的 `memory` 系统手动维护——自动提炼的人格如果错了，比没有更糟糕
- 这是最高层的抽象，也是最容易出错的一层

### 整体节奏

```
Level 0-2 = 地基（事实层）   → 本月实施
Level 3   = 第一层楼（模式层）→ 1-2 个月后评估
Level 4   = 屋顶（人格层）   → 3 个月后评估
```

论文里的凝练过程是**自底向上逐层**的——没有底层就没有高层。先打好地基。

---

## 完成后会有哪些改变

| 场景 | 现在 | 整改后 |
|------|------|--------|
| "昨天说了什么" | 搜到几十个片段，要自己翻 | 直接返回当日 Session 摘要 |
| "关于 API 的讨论" | 搜到一堆相似片段，分不清哪次 | 返回 Event，有标题和结论 |
| "那个时间说的那件事" | 难精确命中 | Level 0 精确到单条消息 |
| "这周做了什么" | 搜不出 | Level 2 Session 按周汇总 |
| 搜索速度 | 700+ 条全量搜 | 层级过滤，每层数据量更少 |

---

## 写在最后

这篇文章写的不是一个"已经做好的"方案，而是一个**正在做的**方案。

TiMem 论文给了我一个很好的理论框架，但理论和落地之间有一条很长的路。从扁平记事本到三层记忆树，需要改脚本、加 cron、调 prompt、写迁移脚本——每一步都不难，但每一步都需要认真对待。

我在 J1900 上跑这个系统，没有 GPU，没有高速网络，只有一颗 10W 的赛扬和 7.6G 内存。但正是这种限制，让每一项优化都有了明确的目标：**用最少的资源，干最多的事。**

改完之后的进展，我会继续写文章记录。

---

## 更新（2026-07-09）：计划执行情况

文章发出来后，我把计划落地了。以下是对照进度。

### 实际执行路径

与计划不同的是，我没有自己写三层 Qdrant 集合，而是**直接用了 TiMem 仓库自带的 FastAPI 服务**（`services/memory_generation_service.py`）。它的 `/generate` 端点接收一段对话内容，自动完成话题分段、LLM 摘要、向量化存储——我把每天的消息按天打包，发给 TiMem 即可。

最终架构变成了这样：

```
飞书 API → feishu_export (13,424 条消息)
                ↓ 按天分组
         TiMem API POST /generate (每天一个 session)
                ↓
         TiMem 内部流水线：
           LangGraph 工作流 → DeepSeek LLM → bge-small-zh → Qdrant
                ↓
         29 条 L1 中文记忆（覆盖 20 天数据）
```

### 踩的坑

1. **DeepSeek API key 认证** — TiMem 的 LLM 适配器有 model 白名单，`deepseek-chat` 不在里面。修：`llm/openai_adapter.py` 加上去
2. **向量维度不匹配** — TiMem 默认 `vector_size: 2048`，bge-small-zh 是 512 维。服务启动检测到不匹配直接 DELETE 重建了 Qdrant 集合，720 条老数据全丢。修：`settings.yaml` 改 `vector_size: 512`，先手动建好集合再启动服务
3. **中文输出** — 默认 system prompt 是英文的，DeepSeek 对中文内容用英文 prompt 也出英文摘要。修：加 `config/zh_cn_prompt.yaml`，`settings.yaml` 设 `language: zh`
4. **Qdrant 408 超时** — 凌晨 3 点和备份 cron 抢 CPU，J1900 4 核被吃满。修：错开时间 + 写入重试
5. **TiMem 调度器未启动** — `scheduler_service.py`（负责 L0→L1→L2 自动凝练）是独立进程，不随 FastAPI 一起启动。正在评估启用时机

### 当前进度

| 层级 | 计划 | 实际 | 状态 |
|------|------|------|------|
| Level 0 原始消息 | Qdrant 独立集合 | TiMem 内部管理 | ✅ 消息已摄入 |
| Level 1 事件摘要 | LLM 每日生成 | TiMem `/generate` 自动处理 | ✅ 29 条 |
| Level 2 会话摘要 | 每周 cron 聚合 | TiMem scheduler 待启动 | ⏸ |
| Level 3+ | 暂缓 | 暂缓 | ⏸ |

每日增量 cron 已配置，04:30 自动跑前一天的数据，走 TiMem API 不走 Qdrant 直写。

文章里写的计划大部分都落地了——虽然中间走了些弯路，但结果是对的。

---

*果壳科技 塔塔*
*2026 年 7 月*
