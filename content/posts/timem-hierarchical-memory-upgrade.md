---
title: "把向量记事本变成记忆树：TiMem 记忆系统的正确打开方式"
description: "从扁平向量存储升级为 TiMem 三层时序记忆树的实战记录——方案、踩坑、结果。"
tags: [timem, ai-memory, qdrant, llm, homelab, deepseek]
categories: 技术
date: 2026-07-09T10:00:00+08:00
image: /goke-tech-blog/img/cover-timem-hierarchy.jpg
draft: false
weight: 4
---

## 缘起：一个"假"的 TiMem

几周前，我在 J1900 家庭服务器上部署了一套名叫"TiMem"的记忆系统。论文说它是"时序层级记忆框架"，有五层结构，能从原始对话自动提炼人格画像。

但实际上我部署了什么？

一个 Qdrant 向量数据库 + 一个嵌入模型 + 两个存/取接口 + 一个 cron 定时往里倒聊天记录。

**说白了，就是一个带语义搜索的记事本。**

720 条记忆，不分层级、不分类别，全部丢进同一个桶里。搜索的时候"一把梭"——全量向量检索，谁跟 query 像就召回谁。没有摘要，没有抽象，没有结构。

论文里说的"时序记忆树"、"层级凝练"、"复杂度感知召回"，一个都没实现。

这就像买了一整套精装书，然后把所有书页拆下来打乱堆在一起，说"我有了一个图书馆"。

**所以这篇文章记录了我们怎么把记事本改造成真正的记忆树——方案、踩坑、结果，全在这里。**

---

## TiMem 论文到底说了什么

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

## 前期方案设计

动手前我们画了三层架构：

```
┌─────────────────────────────────────────────────────┐
│                   TiMem Memory Tree                  │
│                                                       │
│  Level 2:  Sessions          ← LLM 聚合（每周 cron）  │
│    内容: 会话摘要、关键决策、时间范围                   │
│           ↑                                            │
│  Level 1:  Events            ← LLM 摘要（每日 cron）   │
│    内容: 话题标题、摘要、关键词、关联消息数              │
│           ↑                                            │
│  Level 0:  Raw Turns         ← 消息摄入（每日 export）  │
│    内容: 单条消息、用户、时间戳、来源                   │
│                                                       │
│  搜索层: 分析问题 → 选层 → 检索 → 融合返回             │
└─────────────────────────────────────────────────────┘
```

计划分 5 步走：

1. 准备三个 Qdrant 集合
2. 重写飞书导出脚本（每条消息独立嵌入 → Level 0，话题聚类后 LLM 摘要 → Level 1）
3. 新增 consolidation cron（Level 0 → 1 → 2 自动凝练）
4. 改造 MCP 搜索工具（支持层级路由）
5. 老数据迁移

看起来井井有条。但真动手后发现——**TiMem 自己已经有一套完整的服务端架构**，试图绕过它自己写 Qdrant 操作是走弯路。

---

## 实际执行：发现真正的 TiMem

### 走弯路：自己写 Qdrant

一开始我直接写了个 `feishu_to_timem.py`：

- 导出飞书聊天记录到 Markdown
- 按 30 分钟窗口切话题块
- 用 bge-small-zh 嵌入
- 直接写入 Qdrant `timem_memories` 集合

这条路走通了，但有几个问题：

1. **粒度太粗**：一个话题块可能包含多轮对话，搜单条消息时只能返回整个块
2. **没有 LLM 摘要**：存的只是原始文本，没有语义理解
3. **接管了 TiMem 的工作**：TiMem 本身就有 `/generate` 接口，它知道怎么处理消息

### 发现 TiMem 服务端

仔细看了 TiMem 的仓库，发现它根本不是论文配套代码那么简单——它包含：

- **FastAPI 服务** (`services/memory_generation_service.py`)
- **LangGraph 工作流** (`timem/workflows/`)
- **五级记忆系统** (`timem/memory/l1_fragment_memory.py` → `l5_high_level_memory.py`)
- **内置调度器** (`timem/workflows/scheduler_service.py`)

它的 `/generate` 端点接收一段对话内容，自动完成：

1. 话题检测和分段
2. LLM 生成 L1 片段摘要
3. 向量化存储到 `timem_memories` 集合
4. 返回结构化记忆

这才是"正确打开方式"——把消息喂给 TiMem，让它自己处理层级凝练。

### 架构修正

```
┌──────────────────────────────────────────┐
│          修正后的实际架构                    │
│                                            │
│  ┌──────────────┐    ┌────────────────┐   │
│  │ 飞书 API     │ →  │ feishu_export  │   │
│  │ 全量导出     │    │ 13424 条消息    │   │
│  └──────────────┘    └───────┬────────┘   │
│                              ↓             │
│  ┌──────────────┐    ┌────────────────┐   │
│  │ TiMem API    │ ←  │ 按天分组       │   │
│  │ POST /generate│   │ 每天一个 session│   │
│  └──────┬───────┘    └────────────────┘   │
│         ↓                                  │
│  ┌──────────────────────────────────┐      │
│  │ TiMem 内部处理流水线               │      │
│  │ 1. LangGraph 工作流编排           │      │
│  │ 2. LLM (DeepSeek) 生成 L1 摘要   │      │
│  │ 3. bge-small-zh 向量化           │      │
│  │ 4. 存入 Qdrant timem_memories    │      │
│  └──────────────────────────────────┘      │
│         ↓                                  │
│  ┌──────────────┐                          │
│  │ 29 条 L1     │ ← 20 天数据              │
│  │ 中文记忆     │    8,992 条消息           │
│  └──────────────┘                          │
└──────────────────────────────────────────┘
```

---

## 踩坑实录

这条路上坑不少，记录一下，免得后来人再踩。

### 坑 1：DeepSeek API 认证

**现象**：TiMem 调用 DeepSeek 一直报 401/403。

**原因**：TiMem 的 LLM 适配器默认只认 OpenAI 系列的模型名白名单。`deepseek-chat` 不在白名单里。

**解决**：在 `llm/openai_adapter.py` 的 `OPENAI_COMPATIBLE_MODELS` 白名单中加上 `deepseek-chat`。

```python
# llm/openai_adapter.py
OPENAI_COMPATIBLE_MODELS = {
    "gpt-4o", "gpt-4o-mini", "gpt-4-turbo",
    "deepseek-chat",  # ← 加上这一行
}
```

另外要注意 API Key 的有效性。DeepSeek 的 key 如果不绑余额或额度用完，不会给友好的错误提示——直接返回 `401 Unauthorized`。

### 坑 2：向量维度不匹配

**现象**：TiMem 服务启动时报错，Qdrant 集合 `timem_memories` 被重建。

**原因**：TiMem 默认配置 `vector_size: 2048`，但我们用的 bge-small-zh 模型输出 512 维。服务启动时检测到已存在的集合维度不匹配，**直接 DELETE + 重建**，所有历史数据丢失。

**解决**：

1. `config/settings.yaml` 中 `vector_size: 512`
2. 提前用 curl 建好集合，指定 512 维和较长的超时时间
3. **教训：在 Qdrant 里手动建好集合再启动 TiMem，不要让 TiMem 自动创建**

```bash
curl -X PUT "http://localhost:16333/collections/timem_memories" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {"size": 512, "distance": "Cosine"},
    "timeout": 120
  }'
```

### 坑 3：调度器未启动

**现象**：L1 记忆能生成，但 L2（Session 摘要）、L3（每日总结）一直没有出现。

**原因**：TiMem 的 `scheduler_service.py` 是一个独立的 APScheduler 进程，不随 FastAPI 一起启动。需要手动运行或集成到服务入口。

**解决**：在 `start_timem_service.py` 中启动调度器，或者拆成单独的 systemd 服务。

### 坑 4：Qdrant 408 超时

**现象**：并发写入时 Qdrant 返回 408 Request Timeout。

**原因**：J1900 只有 4 核，bge 嵌入在高并发下吃满 CPU，Qdrant 拿不到时间片处理写入请求。

**解决**：
- 将飞书导出 cron（03:00）和 TiMem 导入 cron（04:30）错开时间
- 给 Qdrant 集合设置 `timeout: 120`
- 每次导入只处理一天的数据，避免并发压力

### 坑 5：中文 prompt

**现象**：TiMem 生成的摘要默认是英文。

**原因**：TiMem 的 system prompt 是英文的。DeepSeek 对中文内容用英文 prompt，输出也是英文。

**解决**：创建 `config/zh_cn_prompt.yaml`，把摘要 prompt 改成中文：

```yaml
system_prompt: |
  你是一个记忆摘要助手。请用中文为用户生成对话记忆摘要。
  
  要求：
  1. 用简洁的中文概括对话核心内容
  2. 突出关键决定、结论和后续行动
  3. 保持客观，不添加对话中没有的信息
```

然后在 `settings.yaml` 中启用：
```yaml
prompt:
  custom_prompt_path: config/zh_cn_prompt.yaml
```

---

## 最终成果

经过两天的调试和 20 天数据的全量导入，最终成果如下：

| 指标 | 数值 |
|------|------|
| 有效消息数 | 8,992 条（从 13,424 条导出中解析） |
| 覆盖时间 | 2026-06-20 → 2026-07-09（20 天） |
| L1 记忆数 | 29 条中文摘要 |
| 向量库 | Qdrant 512 维 Cosine 距离 |
| 嵌入模型 | bge-small-zh |
| LLM | DeepSeek V4 Flash（通过 OpenAI 适配器） |
| 每日增量成本 | < ¥0.01/天 |

29 条 L1 记忆覆盖了这 20 天的所有主要话题：qBittorrent 管理、AgsvPT 种子巡检、Hermes 配置、Immich/Jellyfin/Nowen 部署、API key 配置、cron 维护、B 机 WOL、技术笔记撰写等等。

每条记忆都是 LLM 生成的中文摘要——不再是一段原始文本，而是"这段对话在说什么"的语义理解。

---

## 后续计划

L1 已经就位，接下来：

| 层级 | 内容 | 状态 | 计划 |
|------|------|------|------|
| Level 1 | 话题事件 | ✅ 20 天 / 29 条 | 每日增量 cron 04:30 自动导入 |
| Level 2 | 会话摘要 | ❌ 待启动 | TiMem 内置 scheduler 每天 02:00 自动聚合 |
| Level 3 | 每周总结 | ⏸ 暂缓 | 积累 30+ 个 Session 后评估 |
| Level 4+ | 人格画像 | ⏸ 暂缓 | 1-2 个月后评估 |

Level 2 的调度器已经写好了（`scheduler_service.py`），只是还没启动。它每天凌晨 02:00 把当天的 L1 碎片聚合成 L2 Session 摘要。等 TiMem 服务稳定运行两天后再启用。

论文里的凝练过程是**自底向上逐层**的——没有底层就没有高层。先打好地基。

---

## 写在最后

这次整改让我重新理解了 TiMem：**它不是一个数据库，而是一条处理流水线。**

最开始我把它当 Qdrant 用——自己写脚本、自己切话题、自己嵌入。这就像买了一台咖啡机但只用来接热水，真正的价值在于它的研磨和萃取流程。

TiMem 的价值不在于"存"，而在于"炼"：

- **原始消息** → **话题片段**（时间聚类）
- **话题片段** → **记忆摘要**（LLM 提炼）
- **记忆摘要** → **会话总结**（跨事件聚合）
- **会话总结** → **行为模式** → **人格画像**（更高层的抽象）

每一步都需要 LLM 参与——这不是简单的向量相似度能替代的。而 TiMem 的 `/generate` 接口和 LangGraph 工作流正是为这个"炼"的过程设计的。

在 J1900 上跑这套系统，没有 GPU，没有高速网络，只有一颗 10W 的赛扬和 7.6G 内存。DeepSeek API 往返一次要 15-25 秒，生成一条 L1 记忆约 30 秒。20 天数据跑了约 10 分钟。

慢，但能用。

**正是这种限制，让每一项优化都有了明确的目标：用最少的资源，干最多的事。**

---

*果壳科技 塔塔*
*2026 年 7 月*
