---
title: "手把手搭建个人AI记忆系统：Hermes + Hindsight + GBrain 全栈实践"
date: 2026-07-06
draft: false
tags: ["AI", "记忆系统", "知识管理", "Hermes", "Hindsight", "GBrain"]
categories: ["技术实践"]
author: "果壳科技 聂小雨"
summary: "从零搭建个人AI记忆系统的完整指南，涵盖Hermes Agent、Hindsight记忆服务、GBrain知识图谱的部署与整合，实现跨系统语义搜索。"
---

## 写在前面

你有没有想过，和AI助手的对话能像人类一样拥有持久记忆？你上周讨论过的项目方案、上个月定下的技术选型、甚至去年的某个灵感——AI都能记得清清楚楚。

这不是科幻，这是我们正在做的事。

今天分享我们果壳科技团队搭建的个人AI记忆系统，核心技术栈：**Hermes Agent + Hindsight + GBrain**，辅以 Obsidian 知识库和统一的 Embedding 服务。

## 系统架构

整个系统分三层：

```
┌─────────────────────────────────────────────────────────┐
│                    会话与任务层                           │
│   Hermes Agent（对话调度）+ Obsidian（笔记沉淀）          │
├─────────────────────────────────────────────────────────┤
│                    记忆与知识层                           │
│   Hindsight（事实记忆）+ GBrain（知识图谱）               │
├─────────────────────────────────────────────────────────┤
│                    向量计算层                             │
│   Qwen3-Embedding-0.6B（TEI服务，1024维）               │
└─────────────────────────────────────────────────────────┘
```

**关键设计决策**：两个记忆系统共享同一个 Embedding 模型和向量空间，确保跨系统语义搜索的一致性。

## 为什么需要两套记忆系统？

很多人会问：有 Hindsight 不就够了吗？为什么还要 GBrain？

答案是**分工不同**：

- **Hindsight**：擅长事实记忆——谁说了什么、什么时候说的、偏好是什么。它是一个"记忆银行"，存的是对话中提取的事实片段。
- **GBrain**：擅长知识图谱——概念之间的关系、代码结构、时间线。它是一个"第二大脑"，存的是结构化的知识。

打个比方：Hindsight 像你的日记本，记着每天发生的事；GBrain 像你的笔记本，整理着各种知识和关系。两者互补，缺一不可。

## 技术选型：为什么选 Qwen3-Embedding-0.6B？

Embedding 模型是整个系统的基石——它决定了"语义理解"的质量。

我们评估了多个选项：

| 模型 | 维度 | 中文能力 | 内存占用 | 评价 |
|------|------|----------|----------|------|
| bge-small-zh-v1.5 | 512 | 优秀 | ~130MB | 够用但维度偏低 |
| bge-base-zh-v1.5 | 768 | 优秀 | ~400MB | 性能均衡 |
| Qwen3-Embedding-0.6B | 1024 | 优秀 | ~1.5GB | 维度高，表达力强 |
| Qwen3-Embedding-4B | 2560 | 顶级 | ~8GB | 杀鸡用牛刀 |

最终选择 **Qwen3-Embedding-0.6B**，理由：

1. **数据量不大**：总共才 3600 条（GBrain 3032 + Hindsight 569），0.6B 绑绑有余
2. **1024 维够用**：比 768 维多 33% 的表达空间，又不像 4B 那样浪费内存
3. **中文能力强**：Qwen 系列对中文的理解是第一梯队
4. **部署简单**：sentence-transformers 兼容，TEI 一键启动

## 部署实战

### 1. Embedding 服务

```bash
# 下载模型
huggingface-cli download Qwen/Qwen3-Embedding-0.6B --local-dir /home/xiaoyu/models/Qwen3-Embedding-0.6B

# 启动 TEI 服务（systemd）
# 端口 8082，OpenAI 兼容 API
curl http://localhost:8082/health
# → {"status":"ok","model":"Qwen3-Embedding-0.6B","max_length":8192}
```

### 2. GBrain 迁移

```sql
-- 修改向量维度 768 → 1024
UPDATE content_chunks SET embedding = NULL;
ALTER TABLE content_chunks ALTER COLUMN embedding TYPE vector(1024);

-- Python 脚本批量 embed（gbrain reindex 有超时问题，推荐直连 PG）
-- 批量调用 /v1/embeddings API，每批 32 条
```

### 3. Hindsight 迁移

```bash
# 修改 env 配置
HINDSIGHT_API_EMBEDDINGS_PROVIDER=openai
HINDSIGHT_API_EMBEDDINGS_OPENAI_API_KEY=***
HINDSIGHT_API_EMBEDDINGS_OPENAI_MODEL=Qwen3-Embedding-0.6B
HINDSIGHT_API_EMBEDDINGS_OPENAI_BASE_URL=http://localhost:8082/v1

# 重启服务
sudo systemctl restart hindsight-api

# 同样用 Python 脚本重建向量
```

### 4. 最终验证

```bash
# GBrain: 3032/3032 chunks (100%)
# Hindsight: 569/569 memories (100%)
# HNSW 索引已建，搜索响应 <100ms
```

## 踩坑记录

**坑1：gbrain reindex 命令反复超时**

gbrain 内置的 reindex 命令依赖 OpenAI SDK，有内置超时机制。长任务（3000+ chunks）会被 SIGTERM 杀掉。

**解决方案**：绕过 CLI，用 Python 脚本直连 PostgreSQL + 调用 TEI API，分批处理。

**坑2：Hindsight 的 TEI 模式不兼容**

Hindsight 的 TEI 客户端期望调用 `/info` 端点，但我们的 TEI 服务没有这个端点。

**解决方案**：改用 OpenAI 兼容模式，指向 `http://localhost:8082/v1/embeddings`。

**坑3：Hindsight 无 reindex API**

Hindsight 没有暴露重新构建向量的 API 端点。

**解决方案**：直接操作数据库，用 Python 脚本批量更新 embedding 列。

## 成本与收益

**硬件成本**：0 元（全在现有服务器上跑，内存增加约 1GB）

**时间成本**：约 45 分钟完成全量迁移

**收益**：
- 跨系统语义搜索：同一个问题，同时搜到 Hindsight 的事实记忆和 GBrain 的知识图谱
- 中文理解提升：Qwen3 对中文的语义理解明显优于之前的英文模型
- 统一维护：一个 Embedding 服务，一套配置，省心

## 下一步计划

1. **Obsidian 知识库自动化**：每日笔记自动归档、会话自动同步
2. **情报扫描系统**：每日定时扫描 AI/Agent 领域动态
3. **跨系统搜索 API**：封装统一的搜索接口，支持一次查询两个系统

## 写在最后

个人 AI 记忆系统不是什么遥不可及的东西。一台普通的服务器、几个开源项目、一点耐心，就能搭建出属于自己的"第二大脑"。

关键不是技术多复杂，而是**想清楚你要记什么、怎么记、怎么用**。技术只是实现手段，思维方式才是核心。

如果你也在折腾类似的系统，欢迎交流。我们果壳科技一直在探索 AI Agent 的可能性，这只是开始。

---

*本文由果壳科技聂小雨撰写，首发于 GitHub Pages。*
*技术栈：Hermes Agent + Hindsight + GBrain + Qwen3-Embedding-0.6B*
*服务器：Ubuntu 24.04，16核27GB内存*
