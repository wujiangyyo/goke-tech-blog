---
title: "TiMem 记忆管道全自动化：从部署调试到日常无人值守"
description: "修复 TiMem 层级记忆（L2/L3）不自动生成的根因，打通飞书记录→L1导入→L2-L5全链路自动生成。"
tags: [timem, ai-memory, debugging, automation, pipeline, homelab]
categories: ["技术"]
author: "果壳科技 塔塔"
date: 2026-07-10T08:00:00+08:00
draft: false
weight: 5
---

## 背景

上篇文章提到，我把 TiMem 的 L2-L5 层级记忆生成流程跑通了。但上线后遇到一个尴尬的问题：

**每天 04:30 的飞书导入 Cron 把 L1 记忆倒进去了，L2/L3 却纹丝不动。**

查数据库证实：L2 和 L3 始终只有 07-09 之前的数据，07-10 的数据无论怎么等都不生成。

这不是偶发故障，而是架构层面的缺口——**L1 导入管道和 L2-L5 调度器之间没有连线**。

---

## 架构回顾

TiMem 的数据管道分两层：

### 层一：L1 导入（Feishu → 原始片段记忆）

```
飞书记录 → feishu_to_timem_v2.py → POST /generate → core_memories 表 (level='L1')
```

脚本每天 04:30 由 cron 触发：
1. 拉取昨日 03:00 → 今日 03:00 的飞书消息
2. 按 30 分钟间隔做话题聚类
3. 每话题调用 TiMem API 生成一条 L1 片段记忆

### 层二：L2-L5 生成（原始片段 → 层级提炼）

TiMem 内建两个调度器：

| 调度器 | 频率 | 机制 |
|:------:|:----:|:----:|
| `session_memory_scan` | 每 10 分钟 | 扫描 `user_expert_states` 表中的活跃会话，对超时会话生成 L2 |
| `daily_memory_backfill` | 每天 02:00 | 检测昨天缺失的 L2-L5，批量补齐 |

---

## 根因分析

### 问题一：L1 导入不走调度器通道

`session_memory_scan` 依赖 `user_expert_states` 表——这是 Hermes 实时聊天时更新的。但 `feishu_to_timem_v2.py` 调用 `/generate` 只写 `core_memories`，**不写 `user_expert_states`**。

所以无论导入多少 L1，session_memory_scan 都看不见。

### 问题二：每日 backfill 只能补昨天

`daily_memory_backfill` 每天 02:00 运行，检查 **昨天** 的时间窗。

L1 导入在 04:30，比 backfill 晚了 2.5 小时。于是：
- 02:00 backfill 跑：昨天数据还没 L1 → 无任务
- 04:30 L1 导入：数据入库
- **L2-L5 得等到明天 02:00 才能生成**

这就是 **22 小时窗口期** 的由来。

### 问题三：手动 backfill 不检测今天

即使通过 `/scheduler/trigger/backfill` 手动触发，它的 `detect_manual_completion()` 也默认使用 `force_update=False`，只检测 **已完成的时间窗**（昨天、上周、上个月），**跳过当前不完整的时间窗**（今天、本周、本月）。

所以手动触发也无法生成今天缺少的 L3。

---

## 修复方案

三处修改，打通全链路：

### 1️⃣ 新增 `POST /backfill` API 端点

```python
# memory_generation_service.py
class BackfillRequest(BaseModel):
    user_id: str
    expert_id: str
    layers: Optional[List[str]]  # 默认 [L2, L3, L4, L5]

@app.post("/backfill")
async def backfill_endpoint(request, background_tasks):
    background_tasks.add_task(
        backfill_service.backfill_for_user,
        user_id=request.user_id,
        expert_id=request.expert_id,
        layers=request.layers,
    )
    return {"success": True, "message": "Backfill task queued"}
```

使用 `BackgroundTasks` 异步执行，调用方立即收到响应，不阻塞。

### 2️⃣ force_update 改为 True

```python
# scheduled_backfill_service.py
all_missing = await self.catchup_detector.detect_manual_completion(
    user_id=user_id,
    expert_id=expert_id,
    force_update=True  # ← 原来是 False，导致跳过今天
)
```

`force_update=True` 让检测覆盖当前不完整时间窗（今天 L3、本周 L4、本月 L5）。

### 3️⃣ 导入脚本自动触发

```python
# feishu_to_timem_v2.py
def trigger_backfill(user_id: str, expert_id: str) -> bool:
    resp = requests.post(f"{TIMEM_API}/backfill", json={
        "user_id": user_id,
        "expert_id": expert_id,
    }, timeout=30)
    return resp.json().get("success", False)

# 所有话题导入成功后调用
if success > 0:
    trigger_backfill("wujiang", "tata")
```

---

## 修复效果

### 数据对比

| 层级 | 修复前 | 修复后 | 变化 |
|:---:|:------:|:------:|:----:|
| L1 | 46 | 46 | 不变 |
| L2 | 21 | 30 | +9 条 session 记忆 |
| L3 | 21 | **22** | **新增 07-10 日报** |
| L4 | 0 | 2 | 周报首先生成 |
| L5 | 0 | 2 | 月报同时生成 |

### 自动化流程

```
04:30  cron 触发
  └─ feishu_to_timem_v2.py
       ├─ 飞书 API 拉消息
       ├─ 话题聚类 → POST /generate → L1
       └─ POST /backfill
            ├─ catchup_detector 检测缺失
            ├─ MemoryGenerationWorkflow 生成 L2
            ├─ 自动生成 L3（今天日报）
            ├─ 自动生成 L4（本周周报）
            └─ 自动生成 L5（本月月报）
```

以后每日导入 → 全链路自动完成，零人工介入。

---

## 踩坑记录

### 1. BackgroundTasks 的异步问题

FastAPI 的 `BackgroundTasks` 支持 async callable，但 `backfill_for_user` 是 async 方法内调 `asyncio.gather` 的。在测试中发现如果 background task 内部抛异常，日志里能看到但 HTTP 响应不会知道。建议在生产环境加一个任务队列（如 Celery/Redis Queue）。

### 2. force_update=False 是有意为之的设计

TiMem 的 CatchUpDetector 区分 force 和 regular 模式：
- **Regular（默认）**：只补已完成时间窗，保证数据不膨胀
- **Force**：强刷当前不完整窗口，适合手动触发的"立即更新"场景

之前 `backfill_for_user` 用 regular 模式是因为担心 force 模式产生重复数据。实测发现 TiMem 的生成层有幂等性保障（`force_update` 标记区分创建/修改），所以 force 模式是安全的。

### 3. 幂等性和去重

CatchUpDetector 内置去重逻辑（`_deduplicate_tasks`），以 `(user_id, expert_id, layer, session_id/time_window)` 为键去重，优先保留 `force_update=True` 的任务。所以多次触发 backfill 不会产生重复记忆。

---

## 总结

这次修复看起来改动不大——代码层面只改了三个文件、约 60 行——但定位过程涉及理解两层调度器、三个 API 端点、以及数据表之间的关系。

核心教训：**不要把管道两端都设计成"被动等待"模式**。L1 导入是主动的（cron），L2-L5 生成是主动的（cron），但两者之间没有触发联动，全靠"等下一次调度对齐"。加上一根连线后，整个系统才真正闭环。

完整代码见 TiMem 仓库：[github.com/timem-ai/TiMem](https://github.com/timem-ai/TiMem)
