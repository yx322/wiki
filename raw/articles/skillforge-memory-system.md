# SkillForge 记忆系统设计解析

> 基于 docs/memory-system-design.md 整理

---

## 一、设计目标

1. **框架无关**：记忆逻辑不依赖任何特定 Agent 框架
2. **Prefix Checkpoint**：不单独调 LLM，压缩通过 Agent 正常 tool call 完成
3. **向量 + 关键词混合检索**：HNSW + FULLTEXT + RRF 融合排序
4. **LLM 无感知**：阈值以下无额外开销，不膨胀 prompt，不破坏 KV cache

## 二、三层记忆架构

```
┌─────────────────────────────────────────────────┐
│                  Agent Layer                      │
├─────────────────────────────────────────────────┤
│  ConversationMemory      MemoryManager          │
│  (session 级)             (user 级)              │
│  ┌──────────────┐        ┌──────────────┐        │
│  │ 消息追加      │        │ store        │        │
│  │ 滚动压缩      │        │ search (RRF) │        │
│  │ context 获取  │        │ forget       │        │
│  └──────────────┘        └──────────────┘        │
│                                                  │
│  TaskListManager (新增)                           │
│  (user + session 级)                              │
│  ┌──────────────┐                                │
│  │ create       │                                │
│  │ add_item     │                                │
│  │ get          │                                │
│  │ set_done     │                                │
│  └──────────────┘                                │
└─────────────────────────────────────────────────┘
```

| 层 | 表 | 写入方式 | 用途 |
|----|-----|---------|------|
| **对话记忆** | `session_messages` + `session_checkpoints` | 实时追写 + 滚动压缩 | 最近对话上下文 |
| **长期记忆** | `memories` | Agent 主动调用 + 压缩时自动提取 | 跨会话知识积累 |
| **任务记忆** | `tasks` + `task_items` | LLM 经 memento 工具操作 | 多步任务执行状态 |
| **会话记忆** | `agno_sessions` | 框架自动写入 | 对话历史持久化 |

## 三、对话记忆（ConversationMemory）

### 3.1 Prefix Checkpoint 机制

```
消息追加 → 累积到阈值(100条) → 下一个用户提问时注入尾提示词
  → Agent 一次 turn 完成：
    1. 回答用户问题
    2. 调用 memory_store 提取长期记忆
    3. 调用 memory_checkpoint 压缩为摘要
  → checkpoint 不可变，对 KV cache 友好
```

### 3.2 两种记忆的分工

| 记忆类型 | 触发方式 | 谁生成内容 | 用途 |
|----------|----------|-----------|------|
| **长期记忆**（memory_store） | **主动**：LLM 判断"值得记住"时调用 | LLM 决定存什么 | 跨会话知识，语义检索 |
| **短期记忆**（memory_checkpoint） | **被动**：阈值触发，prompt 注入 | LLM 生成对话摘要 | KV cache 优化，上下文压缩 |

### 3.3 数据隔离

- **对话记忆**：按 `user_id + session_id` 双键隔离
- **长期记忆**：只按 `user_id` 隔离，跨 session 检索

### 3.4 为什么不用 Agno 的滑动窗口

Agno 的 `add_history_to_context` 每次从 DB 读最近 N 条消息，N 不断变化 → **缓存完全失效**。

Checkpoint 是**不可变的**，一旦写入永远不变 → 可永久缓存。

## 四、长期记忆（MemoryManager）

### 4.1 核心方法

| 方法 | 用途 |
|------|------|
| `store(content, category, importance, tags)` | 存储记忆 |
| `search(query, top_k)` | HNSW + BM25 + RRF 混合检索 |
| `forget(memory_id)` | 删除记忆 |
| `list_all(category, limit)` | 列出记忆（管理用途）|
| `get_stats()` | 统计信息（仪表盘）|

### 4.2 Embedding 生成

- **写入侧**：PostgreSQL plpython3u 触发器在 INSERT 时调 DashScope 自动生成（1024 维）
- **查询侧**：Python 计算（PG 无内建 query embedding）
- **降级**：无 API KEY 时查询降级为纯全文检索

### 4.3 混合检索

```
HNSW 向量检索（cosine）
    +
BM25 全文检索（jieba 分词）
    ↓
RRF 融合排序（k=60）
```

## 五、Agent 工具注入

| 工具 | 职责 | 谁决定 |
|------|------|--------|
| `memory_store` | 长期记忆提取 | LLM 主动判断 |
| `memory_checkpoint` | 短期记忆压缩 | 记忆系统（阈值触发）|
| `memory_search` | 记忆检索 | LLM 主动判断 |

## 六、Agent 框架集成

### 6.1 _AgnoAgentWrapper 三步调用

```python
wrapper.run(user_message)
  ① memory.push_user_message(session_id, user_message)   # 缓冲
  ② memory.get_context(session_id)                        # checkpoint + DB + pending + 尾提示词
  ③ agent.additional_input = context                      # 注入到 user message 之前
  ④ agent.run()                                            # 模型生成
  ⑤ memory.write_assistant_message(session_id, content)   # 写入 DB
```

### 6.2 迁移到其他框架

只需重写 `_AgnoAgentWrapper` 的适配层。MemoryManager 和 MemoryTools 完全复用。

## 七、PostgreSQL Schema

### memories 表

```sql
CREATE TABLE memories (
    id bigserial PRIMARY KEY,
    user_id text,
    content text,
    category text CHECK (category IN ('preference','fact','decision','context')),
    importance int CHECK (importance BETWEEN 1 AND 5),
    tags jsonb,
    embedding vector(1024),       -- plpython3u 触发器生成
    source_session text,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz
);
-- 索引：HNSW (cosine) + BM25 (jieba) + btree (user_id, category, created_at)
```

### session_messages 表

```sql
CREATE TABLE session_messages (
    id bigserial PRIMARY KEY,
    user_id text,
    session_id text,
    role text,          -- user / assistant
    content text,
    checkpoint_id bigint,
    created_at timestamptz DEFAULT now()
);
```

### session_checkpoints 表

```sql
CREATE TABLE session_checkpoints (
    id bigserial PRIMARY KEY,
    user_id text,
    session_id text,
    checkpoint_index int,
    summary text,           -- 不可变
    message_count int,
    created_at timestamptz
);
```
