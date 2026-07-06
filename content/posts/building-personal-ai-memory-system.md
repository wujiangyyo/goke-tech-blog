---
title: "手把手搭建个人AI记忆系统：Hermes + Hindsight + GBrain 全栈实践"
date: 2026-07-06
draft: false
tags: ["AI", "记忆系统", "知识管理", "Hermes", "Hindsight", "GBrain", "Embedding"]
categories: ["技术实践"]
author: "果壳科技 聂小雨"
summary: "从零搭建个人AI记忆系统的完整指南。我们用Hermes Agent做对话调度，Hindsight存事实记忆，GBrain建知识图谱，再用Qwen3-Embedding-0.6B统一三个系统的向量空间。踩坑无数，全部记录。"
---

## 写在前面

你有没有想过，和AI助手的对话能像人类一样拥有持久记忆？你上周讨论过的项目方案、上个月定下的技术选型、甚至去年的某个灵感——AI都能记得清清楚楚。

这不是科幻，这是我们果壳科技正在做的事。

我们花了两天时间，搭建了一套完整的个人AI记忆系统。今天把全过程分享出来，包括架构设计、技术选型、部署细节，以及踩过的每一个坑。

核心数据：**3601条记忆、1024维向量空间、45分钟完成迁移、零额外硬件成本。**

## 一、系统架构

### 1.1 整体设计

```
┌─────────────────────────────────────────────────────────┐
│                    会话与任务层                           │
│   Hermes Agent（对话调度）+ Obsidian（笔记沉淀）          │
├─────────────────────────────────────────────────────────┤
│                    记忆与知识层                           │
│   Hindsight（事实记忆）+ GBrain（知识图谱）               │
├─────────────────────────────────────────────────────────┤
│                    向量计算层                             │
│   Qwen3-Embedding-0.6B（TEI服务，端口8082）              │
└─────────────────────────────────────────────────────────┘
```

三个层次各司其职：
- **会话层**负责日常对话、任务调度、笔记归档
- **记忆层**负责长期记忆的存储、检索、关联
- **向量层**负责把自然语言转换成计算机能理解的数学表示

### 1.2 为什么需要两套记忆系统？

很多人会问：有 Hindsight 不就够了吗？为什么还要 GBrain？

答案是**分工不同**：

| 系统 | 擅长 | 类比 | 数据量 |
|------|------|------|--------|
| Hindsight | 事实记忆：谁说了什么、什么时候、偏好习惯 | 日记本 | 569条 |
| GBrain | 知识图谱：概念关系、代码结构、时间线 | 笔记本 | 3032个chunk |

Hindsight 从对话中提取事实片段，比如"老板喜欢喝美式咖啡""项目截止日期是下周三"。它是一个"记忆银行"，存的是结构化的事实。

GBrain 管理知识网络，比如"Qwen3-Embedding是阿里通义千问团队开发的""Hindsight用PostgreSQL存储向量"。它是一个"第二大脑"，存的是概念和关系。

两者互补，缺一不可。单一系统无法同时满足"记住对话内容"和"理解知识结构"的需求。

## 二、技术选型：为什么选 Qwen3-Embedding-0.6B？

### 2.1 什么是Embedding？

Embedding 是把自然语言转换成向量（一串数字）的过程。比如"苹果"可能被表示成 `[0.12, -0.34, 0.56, ...]`，维度越高，表达能力越强。

两个句子的向量越接近，说明它们的语义越相似。这就是"语义搜索"的基础——不依赖关键词匹配，而是理解你真正想问什么。

### 2.2 选型过程

我们评估了多个中文Embedding模型：

| 模型 | 维度 | C-MTEB评分 | 内存占用 | 特点 |
|------|------|-----------|----------|------|
| bge-small-zh-v1.5 | 512 | 61.59 | ~130MB | 轻量，够用但维度偏低 |
| bge-base-zh-v1.5 | 768 | 64.89 | ~400MB | 性能均衡 |
| **Qwen3-Embedding-0.6B** | **1024** | **66.33** | **~1.5GB** | **维度高，表达力强** |
| Qwen3-Embedding-4B | 2560 | 70+ | ~8GB | 性能顶级，但杀鸡用牛刀 |

### 2.3 最终选择：Qwen3-Embedding-0.6B

理由很简单：

**1. 数据量决定了不需要大模型**

我们的数据量是 3600 条（GBrain 3032 chunks + Hindsight 569 memories）。这个量级，0.6B 的模型绑绑有余。4B 模型的提升在小数据集上几乎感知不到，但内存占用多出 5 倍。

**2. 1024维是甜蜜点**

768维是很多模型的默认选择，但1024维多出33%的表达空间，在处理中文这种信息密度高的语言时更有优势。2560维虽然更强，但对于我们的数据量来说是浪费。

**3. Qwen系列的中文理解是第一梯队**

阿里的通义千问团队在中文NLP上的积累不用多说。Qwen3-Embedding在C-MTEB（中文语义理解基准测试）上得分66.33，比bge系列高3-5分。

**4. 部署简单**

模型兼容 sentence-transformers 格式，用TEI（Text Embeddings Inference）一键启动，OpenAI兼容API，任何支持OpenAI embedding的工具都能直接对接。

## 三、部署实战

### 3.1 部署Embedding服务

**第一步：下载模型**

```bash
# 从 HuggingFace 下载（国内用 hf-mirror.com 加速）
HF_ENDPOINT=https://hf-mirror.com huggingface-cli download \
  Qwen/Qwen3-Embedding-0.6B \
  --local-dir /home/xiaoyu/models/Qwen3-Embedding-0.6B
```

模型大小约 1.2GB，下载完成后验证：

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('/home/xiaoyu/models/Qwen3-Embedding-0.6B')
embedding = model.encode(["测试文本"])
print(f"维度: {embedding.shape}")  # 应该是 (1, 1024)
```

**第二步：启动TEI服务**

我们用systemd管理TEI服务，确保开机自启：

```bash
# 创建 systemd service
cat > /etc/systemd/system/tei-embedding.service << 'EOF'
[Unit]
Description=TEI Embedding Service
After=network.target

[Service]
Type=simple
User=xiaoyu
ExecStart=/usr/local/bin/tei \
  --model-id /home/xiaoyu/models/Qwen3-Embedding-0.6B \
  --port 8082 \
  --max-client-batch-size 32
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 启动并设置开机自启
sudo systemctl daemon-reload
sudo systemctl enable tei-embedding
sudo systemctl start tei-embedding
```

**第三步：验证服务**

```bash
# 健康检查
curl http://localhost:8082/health
# → {"status":"ok","model":"Qwen3-Embedding-0.6B","max_length":8192}

# 测试embedding
curl -X POST http://localhost:8082/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen3-Embedding-0.6B", "input": "你好世界"}'
```

响应中会返回1024个浮点数，这就是"你好世界"的向量表示。

### 3.2 GBrain迁移：768维→1024维

GBrain是一个知识图谱系统，之前用的是bge-base-en-v1.5模型（768维）。我们需要把所有向量从768维迁移到1024维。

**步骤一：备份数据库**

```bash
pg_dump -U postgres -d gbrain > /tmp/gbrain-backup-$(date +%Y%m%d).sql
```

这一步千万别省。向量迁移是破坏性操作，出问题只能回滚。

**步骤二：修改向量维度**

```sql
-- 连接数据库
psql -U postgres -d gbrain

-- 清除旧向量（维度不匹配，必须先清空）
UPDATE content_chunks SET embedding = NULL WHERE embedding IS NOT NULL;

-- 修改列类型
ALTER TABLE content_chunks ALTER COLUMN embedding TYPE vector(1024);
```

**步骤三：批量重建向量**

这里是第一个坑。GBrain自带的 `gbrain reindex` 命令有内置超时机制，3000+条数据的任务会被SIGTERM杀掉。我们尝试了多次，每次都在416条左右被杀。

最终方案：绕过CLI，用Python脚本直连PostgreSQL + 调用TEI API：

```python
import psycopg2
import requests
import json

# 连接数据库
conn = psycopg2.connect(host='localhost', dbname='gbrain', user='postgres')
cur = conn.cursor()

# 获取需要embed的chunks
cur.execute("SELECT id, chunk_text FROM content_chunks WHERE embedding IS NULL ORDER BY id")
rows = cur.fetchall()

# 批量调用TEI API
api_url = "http://localhost:8082/v1/embeddings"
model = "Qwen3-Embedding-0.6B"
batch_size = 32

for i in range(0, len(rows), batch_size):
    batch = rows[i:i+batch_size]
    texts = [r[1][:2000] for r in batch]  # 截断过长文本
    ids = [r[0] for r in batch]
    
    resp = requests.post(api_url, json={"model": model, "input": texts}, timeout=120)
    if resp.status_code == 200:
        embeddings = [item['embedding'] for item in resp.json()['data']]
        for chunk_id, emb in zip(ids, embeddings):
            cur.execute(
                "UPDATE content_chunks SET embedding = %s::vector WHERE id = %s",
                (json.dumps(emb), chunk_id)
            )
        conn.commit()
        print(f"进度: {min(i+batch_size, len(rows))}/{len(rows)}")

# 创建HNSW索引（加速向量搜索）
cur.execute("""
    CREATE INDEX idx_chunks_embedding_hnsw 
    ON content_chunks USING hnsw (embedding vector_cosine_ops) 
    WITH (m=16, ef_construction=200)
""")
conn.commit()
```

这个脚本跑完后，GBrain的3032个chunks全部迁移到1024维空间。

### 3.3 Hindsight迁移：ONNX→共享TEI

Hindsight之前用的是本地ONNX模型（multilingual-e5-small，384维）。我们改成指向共享的TEI服务。

**步骤一：修改配置文件**

编辑 `/home/xiaoyu/.hindsight/profiles/hermes.env`：

```bash
# 注释掉原来的本地ONNX配置
# HINDSIGHT_API_EMBEDDINGS_PROVIDER=local
# HINDSIGHT_API_EMBEDDINGS_ONNX_MODEL_ID=intfloat/multilingual-e5-small

# 新增OpenAI兼容配置
HINDSIGHT_API_EMBEDDINGS_PROVIDER=openai
HINDSIGHT_API_EMBEDDINGS_OPENAI_API_KEY=not-needed
HINDSIGHT_API_EMBEDDINGS_OPENAI_MODEL=Qwen3-Embedding-0.6B
HINDSIGHT_API_EMBEDDINGS_OPENAI_BASE_URL=http://localhost:8082/v1
```

注意：Hindsight的TEI模式不兼容我们的服务（它期望调用 `/info` 端点），所以用OpenAI兼容模式。

**步骤二：修改向量维度**

```sql
psql -U postgres -d hindsight

-- 清除旧向量
UPDATE memory_units SET embedding = NULL WHERE embedding IS NOT NULL;

-- 修改维度
ALTER TABLE memory_units ALTER COLUMN embedding TYPE vector(1024);
```

**步骤三：重启服务**

```bash
sudo systemctl restart hindsight-api
```

验证日志中出现：
```
Embeddings: OpenAI provider initialized (model: Qwen3-Embedding-0.6B, dim: 1024)
```

**步骤四：重建向量**

Hindsight没有暴露reindex API，同样用Python脚本直连PG操作：

```python
import psycopg2
import requests
import json

conn = psycopg2.connect(host='localhost', dbname='hindsight', user='postgres')
cur = conn.cursor()

cur.execute("SELECT id, text FROM memory_units ORDER BY id")
rows = cur.fetchall()

# 同样的批量embed逻辑...
# 569条记忆，约2分钟完成
```

### 3.4 最终验证

```bash
# GBrain状态
psql -U postgres -d gbrain -c \
  "SELECT count(*) as embedded, 
   (SELECT count(*) FROM content_chunks) as total
   FROM content_chunks WHERE embedding IS NOT NULL;"
# embedded | total 
# ----------+-------
#      3032 |  3032

# Hindsight状态
psql -U postgres -d hindsight -c \
  "SELECT count(*) as embedded,
   (SELECT count(*) FROM memory_units) as total
   FROM memory_units WHERE embedding IS NOT NULL;"
# embedded | total 
# ----------+-------
#       569 |   569

# 搜索测试
curl http://127.0.0.1:9092/v1/default/banks/hermes/memories/recall \
  -H "Content-Type: application/json" \
  -d '{"query": "老板的联系方式", "limit": 3}'
```

## 四、踩坑记录

### 坑1：gbrain reindex 命令反复超时

**现象**：`gbrain reindex --markdown` 每次跑到416条左右就被SIGTERM杀掉。

**原因**：gbrain内置的OpenAI SDK有超时机制，300秒超时对3000+条数据不够用。

**解决方案**：绕过CLI，用Python脚本直连PostgreSQL + 调用TEI API。好处是：
- 没有超时限制
- 可以精确控制batch size
- 出错可以精确定位到哪条数据

### 坑2：Hindsight的TEI模式不兼容

**现象**：配置 `HINDSIGHT_API_EMBEDDINGS_PROVIDER=tei` 后，服务启动报错：
```
RuntimeError: Failed to connect to TEI server at http://localhost:8082: 
Client error '404 Not Found' for url 'http://localhost:8082/info'
```

**原因**：Hindsight的TEI客户端期望调用 `/info` 端点获取模型信息，但我们的TEI服务没有这个端点。

**解决方案**：改用OpenAI兼容模式，指向 `http://localhost:8082/v1/embeddings`。

### 坑3：Hindsight无reindex API

**现象**：想通过API触发向量重建，但找不到对应的端点。

**原因**：Hindsight设计上不提供reindex API，它期望embedding在retain时自动计算。

**解决方案**：直接操作数据库，用Python脚本批量更新embedding列。虽然粗暴，但有效。

### 坑4：env文件的注释行干扰

**现象**：用sed替换 `HINDSIGHT_API_EMBEDDINGS_PROVIDER=local` 时，把注释掉的同名行也改了。

**原因**：grep/sed默认不区分注释行和有效行。

**解决方案**：替换时用 `^HINDSIGHT_API_EMBEDDINGS_PROVIDER=local$` 精确匹配（不带 `#` 前缀）。

## 五、成本与收益

### 5.1 成本

| 项目 | 成本 |
|------|------|
| 硬件 | 0元（现有服务器，内存增加约1GB） |
| 软件 | 0元（全部开源） |
| 时间 | 约45分钟完成全量迁移 |
| 人力 | 1人（AI助手自动执行） |

### 5.2 收益

**跨系统语义搜索**

同一个问题，同时搜到Hindsight的事实记忆和GBrain的知识图谱。比如问"老板喜欢什么"，Hindsight返回"喜欢喝美式咖啡"，GBrain返回"果壳科技创始人，经营AI Agent业务"。

**中文理解提升**

Qwen3对中文的语义理解明显优于之前的英文模型。测试中，搜索"服务器配置"能正确匹配到"Ubuntu 24.04, 16核27G"这样的技术细节，而旧模型可能会漏掉。

**统一维护**

一个Embedding服务，一套配置，两个系统共享。以后升级模型只需要改一个地方。

## 六、下一步计划

1. **Obsidian知识库自动化**：每日笔记自动归档、会话自动同步到Obsidian
2. **情报扫描系统**：每日定时扫描AI/Agent领域动态，自动生成简报
3. **跨系统搜索API**：封装统一的搜索接口，支持一次查询同时检索Hindsight和GBrain
4. **模型升级路径**：当数据量增长到1万条以上时，考虑升级到Qwen3-Embedding-4B

## 七、写在最后

个人AI记忆系统不是什么遥不可及的东西。一台16核27G的普通服务器、几个开源项目、一点耐心，就能搭建出属于自己的"第二大脑"。

关键不是技术多复杂，而是**想清楚你要记什么、怎么记、怎么用**。技术只是实现手段，思维方式才是核心。

我们果壳科技一直在探索AI Agent的可能性，这只是开始。如果你也在折腾类似的系统，欢迎交流。

---

*本文由果壳科技聂小雨撰写，首发于GitHub Pages。*

*技术栈：Hermes Agent + Hindsight + GBrain + Qwen3-Embedding-0.6B*

*服务器：Ubuntu 24.04，16核27GB内存*

*完整部署脚本和配置文件见 [GitHub仓库](https://github.com/wujiangyyo/goke-tech-blog)*
