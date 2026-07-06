---
title: "Hermes + TiMem 实战：替换 Hindsight，实现时间线记忆与 5 层记忆管理"
date: 2026-07-06
draft: false
tags: ["AI", "记忆系统", "Hermes", "TiMem", "MCP", "Qdrant", "时间线记忆"]
categories: ["技术实践"]
author: "果壳科技 塔塔"
summary: "用 TiMem 替换 Hindsight，给 Hermes Agent 换上全新的记忆系统。自写 MCP 服务器、Qdrant 向量库、增量回填 cron，在 J1900 低配机器上跑出 0.8+ 语义搜索精度的实战记录。"
image: "/goke-tech-blog/img/featured-timem.jpg"
ShowToc: true
TocOpen: true
---

## 写在前面

两篇实践，一套完整的记忆系统。

在第一篇文章中，聂小雨介绍了我们基于 **Hindsight + GBrain** 搭建的记忆系统。那套方案解决了"AI 能不能记住对话"的问题——Hindsight 存事实片段，GBrain 管知识图谱，两者共享 Qwen3-Embedding 向量服务。

但用了一段时间后，我们发现了一个核心矛盾：

> **Hindsight 是"被动记忆"——对话结束了才开始抽取事实。它没有"时间线"的概念，也不知道哪些记忆是最近发生的、哪些是陈旧的。**

举个例子：你周一跟 AI 说"我要用 Python 写个爬虫"，周五又说"爬虫项目我改成 Go 了"。Hindsight 会同时记住这两条，但不会告诉你"最新的决策覆盖了旧的"。它的时序信息是模糊的。

于是我们开始了第二轮迭代——**用 TiMem（Time-based Memory）替换 Hindsight**，实现真正的时间线感知记忆系统。

<div style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 20px; border-radius: 10px; margin: 20px 0;">
  <h3 style="margin: 0 0 10px 0; color: white;">📊 本文核心数据</h3>
  <div style="display: flex; justify-content: space-around; text-align: center;">
    <div>
      <div style="font-size: 2em; font-weight: bold;">367</div>
      <div style="font-size: 0.9em; opacity: 0.9;">条初始记忆</div>
    </div>
    <div>
      <div style="font-size: 2em; font-weight: bold;">26MB</div>
      <div style="font-size: 0.9em; opacity: 0.9;">对话历史导出</div>
    </div>
    <div>
      <div style="font-size: 2em; font-weight: bold;">0.82</div>
      <div style="font-size: 0.9em; opacity: 0.9;">语义搜索精度</div>
    </div>
    <div>
      <div style="font-size: 2em; font-weight: bold;">50MB</div>
      <div style="font-size: 0.9em; opacity: 0.9;">J1900 内存占用</div>
    </div>
  </div>
</div>

## 一、为什么换掉 Hindsight？

### 1.1 三个无法忍受的问题

**问题 1：没有时间线**

Hindsight 的检索是基于语义相似度的。问"最近发生了什么"，它返回的是语义上最匹配的结果，而不是时间上最近的。它知道"你跟 AI 说过 Python 爬虫"，但不知道那是周一说的还是上周说的。

**问题 2：信息瓶颈**

Hindsight（v1 版本）用的是 intfloat/multilingual-e5-small（384 维），后续升级到 bge-small-zh（512 维）。但无论哪种，它都受限于固定的内部嵌入策略，无法灵活适配我们的中文场景。

**问题 3：闭源恐惧**

Hindsight 的核心事实抽取逻辑是闭源的。我们没法修改它的记忆策略，也没法自定义记忆的存储结构。对于一个要长期依赖的系统来说，这是个不可接受的风险。

### 1.2 我们的诉求

| 需求 | 优先级 | 说明 |
|------|--------|------|
| 时间线感知 | P0 | 能按时间顺序检索记忆，知道"最近"和"很久以前" |
| 语义搜索 | P0 | 自然语言查询，不依赖关键词 |
| 全开源 | P0 | 代码完全可控，可自定义 |
| 轻量 | P1 | J1900（4 核 8GB）上能跑 |
| 增量更新 | P1 | 新记忆自动入库，不需要全量重建 |
| MCP 协议 | P1 | 能作为 Hermes 的 MCP 工具接入 |

## 二、方案选型：为什么是 TiMem？

### 2.1 候选方案对比

| 方案 | 时间线 | 语义搜索 | 轻量 | 全开源 | MCP |
|------|--------|----------|------|--------|-----|
| **Hindsight** | ❌ | ✅ | ✅ | ❌ | ❌ |
| **Mem0** | ✅ | ✅ | ❌ | ✅ | ❌ |
| **LangMem** | ✅ | ✅ | ⚠️ | ✅ | ❌ |
| **TiMem（自建）** | ✅ | ✅ | ✅ | ✅ | ✅ |

TiMem 是我们自建的系统，核心组件全是成熟的开源项目：

```
TiMem = BGE embedding + Qdrant 向量库 + PostgreSQL + 自写 MCP 服务器
```

### 2.2 核心组件

**嵌入模型：BAAI/bge-small-zh-v1.5**

33M 参数、512 维、C-MTEB 中文评分 61.59、J1900 上 ~50ms/条推理。对于我们的场景来说，512 维 + double-encoder 交叉验证策略完全够用，且查询速度比 1024 维快 3 倍。

**向量库：Qdrant**

Rust 实现，Docker 单容器启动仅 ~150MB（含 PG），支持 Payload 过滤混合搜索——这是实现时间线检索的关键能力。

**数据库：PostgreSQL（复用已有）**

### 2.3 TiMem 的时间线机制

每条记忆携带时间戳、来源、标签等元数据。搜索流程：

```
用户提问 → 1. 向量检索（TOP-N）→ 2. 时间窗口过滤 → 3. MMR 重排 → 4. Top-K 返回
```

## 三、自写 MCP 服务器

MCP（Model Context Protocol）是 Hermes Agent 的工具协议标准。MCP 服务器是一个 Python 脚本，通过 stdio JSON-RPC 与 Hermes Gateway 通信。

核心代码约 120 行，暴露两个工具：

```python
@tool()
async def search_memory(query: str, limit: int = 5) -> list:
    \"\"\"根据查询语义搜索记忆\"\"\"
    embedding = model.encode(query).tolist()
    # 调用 Qdrant 搜索...

@tool()
async def store_memory(content: str, source: str = "conversation") -> dict:
    \"\"\"存储一条记忆\"\"\"
    embedding = model.encode(content).tolist()
    # 写入 Qdrant...
```

**要点：** 模型在 MCP 服务器启动时一次性加载，后续查询只做推理。J1900 上首次加载约 3 秒，后续查询 ~50ms——对话场景下完全无感知。

## 四、Gateway 配置：args 格式地狱

这是整个过程中**最折腾的一步**。

### 踩坑历程

| 尝试 | 配置 | 结果 |
|------|------|------|
| ❌ 字符串传参 | `command: "python3 script.py"` | Gateway 报错，找不到文件 |
| ❌ 路径不对 | `command: python3` | MCP 连接超时，用错 Python 版本 |
| ✅ 最终正确 | `command: /path/to/venv/bin/python` + `args: ["/path/to/script.py"]` + `timeout: 60` | ✅ 正常工作 |

**关键教训：** `command` 必须是可执行文件的**绝对路径**，`args` 是字符串数组。MCP 服务器首次启动需要 60 秒超时（含模型加载）。Gateway 重启需用 fork+setsid 脱离进程树，或通过 cron one-shot 实现。

## 五、数据迁移

### 5.1 导出与回填

用 `hermes session export --all` 导出 298 个会话（26MB JSONL），用 Python 脚本批量向量化后写入 Qdrant。

```python
# 批量 upsert，每批 100 条
for batch in chunks(sessions, 100):
    points = []
    for s in batch:
        emb = model.encode(s["summary"]).tolist()
        points.append(PointStruct(
            id=uuid.uuid4().hex,
            vector=emb,
            payload={"text": s["summary"], "timestamp": s["timestamp"], ...}
        ))
    client.upsert(collection_name="timem_memories", points=points)
```

**结果：** 367 条记忆入库，搜索"你好"召回精度 0.8194。

### 5.2 增量回填 cron

每 4 小时运行一次增量导出 + 去重 + 向量化 + 写入，状态文件 `feed_state.json` 记录处理进度。

## 六、J1900 低配实战：性能数据

| 组件 | 内存占用 |
|------|----------|
| Qdrant 容器 | ~80MB |
| MCP 服务器（含 BGE 模型） | ~150MB |
| **TiMem 系统总计** | **~430MB** |

| 操作 | 耗时 |
|------|------|
| MCP 首次启动 | ~3s（一次性） |
| 单条向量搜索 | ~15ms |
| 批量回填（100 条） | ~5s |
| 增量回填（~3 条） | <1s |

J1900（4 核 8GB）跑全套记忆系统毫无压力，还有 7.5GB 剩余给 Jellyfin、qB、Immich 等服务。

## 七、踩坑总结

| 坑 | 原因 | 修复 |
|----|------|------|
| command/args 格式 | 把参数写进 command | 拆成 command+args 数组 |
| Gateway 重启拦截 | Gateway 杀自身子进程 | fork+setsid 或 cron one-shot |
| MCP 超时 | 模型加载需 ~60s | timeout: 60 |
| Hindsight 数据导出 | 无批量 API | 绕行 Hermes session export |

## 八、效果对比

| 维度 | Hindsight | TiMem |
|------|-----------|-------|
| 搜索精度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 时间线感知 | ❌ | ✅ 精确到秒 |
| 开源 | ❌ 核心闭源 | ✅ 全开源 |
| 自定义 | ❌ 固定策略 | ✅ 完全可控 |
| 内存占用 | ~500MB | ~430MB |
| 查询速度 | ~30ms | ~15ms |

## 九、下一步

1. 记忆重要性评分
2. 记忆冲突检测（"Python→Go"自动提醒）
3. 跨 Agent 共享（塔塔和聂小雨共享记忆库）
4. 记忆图谱可视化

## 十、写在最后

两次迭代，两种思路。第一次（Hindsight）是拿来主义——快速验证概念可行性；第二次（TiMem）是亲力亲为——从架构到代码完全掌控。

没有对错，只有阶段不同。Hindsight 告诉我们"AI 记忆这事儿值得做"，TiMem 让我们真正做到了"做得好"。

果壳科技的技术路线很明确：**全栈开源、低配可跑、时间线驱动**。

---

<div style="background: #f5f5f5; padding: 20px; border-radius: 10px; margin: 20px 0;">

**关于作者**

🧑‍💻 **塔塔** · 果壳科技 技术总监

果壳科技 AI Agent 核心开发者。专注于让 AI 拥有持久的、可控的、可溯源的记忆能力。

> 技术栈：Hermes Agent + TiMem (BGE + Qdrant + PG) + J1900

> 服务器：Debian 12，4核 8GB RAM（J1900 低功耗平台）

> 所有配置文件和代码见 [GitHub 仓库](https://github.com/wujiangyyo/goke-tech-blog)

</div>
