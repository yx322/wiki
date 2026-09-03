---
title: Agent 记忆系统的 KV 落地
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, agent, memory, architecture]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# Agent 记忆系统的 KV 落地

AI Agent 应用（智能体记忆库、长短期上下文管理、多轮对话历史检索）的读写模式高度固定——对话历史按时间线扫描、状态快照精确点查、标签实体二级索引。纯 KV 的 [[composite-key-encoding]] 直接映射这三种模式。

## 固定读写模式

| 模式 | 业务场景 | Key 编码 | 查询方式 |
|------|---------|---------|---------|
| **时间线扫描** | 加载某 session 的最近 N 条对话 | `mem:{agent_id}:{session_id}:{ts_be}:{msg_id}` | 前缀 Seek + 迭代器 |
| **精确点查** | 读取 Agent 的 System Prompt | `agent:profile:{agent_id}:{memory_type}` | 单次 Get |
| **二级索引** | 按关键词检索关联对话片段 | `idx:tag:{entity}:{agent_id}:{ts}` → `{session_id}:{msg_id}` | 前缀扫描索引 ID，回表点查 |

## 对话历史倒序加载

LLM 加载最近对话时需要倒序（最新消息优先）。两种实现：

1. `SeekForPrev` 反向迭代器（Fjall 支持）
2. 时间戳编码为 `u64::MAX - timestamp`，使最新消息的 Key 字典序最小，正向迭代器自然倒序

后者兼容所有只支持正向扫描的 KV 引擎。

## 热冷数据天然分层

Agent 对话的访问模式——最近对话被每轮反复读取（热数据），历史对话数月不碰（冷数据）——与 LSM-Tree 的物理结构天然契合：

- 热数据驻留 MemTable + OS Page Cache（内存级延迟）
- 冷数据自动沉降到 SSTable（SSD 存储）
- 无需手动分层或配置缓存策略

## 与 ORM 的 CPU 开销对比

LangChain/LlamaIndex 等框架在存储之上包裹了 ORM + 连接池 + SQL 解析层。读取一段聊天记录要经过：

SQL 字符串解析 → 查询计划生成 → 进程/线程锁竞争 → 数据在网络驱动和应用层之间反序列化

纯 KV 的读路径是：

前缀匹配 → 磁盘/缓存连续字节直接交给反序列化器

在 LLM 每轮需拼装数万 Token 上下文的场景下，省去的 ORM 开销累积可观。

## 向量检索的边界

纯 KV 的 Key 编码能模拟 Redis 的所有数据结构，但**无法在 Key 层面做向量相似度搜索**——LSM-Tree 的字典序排序对高维向量无意义。

Agent 应用中向量检索（RAG）是独立的架构层：

- **嵌入式向量索引 + KV 存储分离**：引入轻量向量索引库（`hnsw-rs`、`faiss-rs`），启动时加载到内存。向量索引返回 ID → KV 点查原文。零外部依赖。
- **原生多模型引擎**：[[surrealdb]] 在 [[surrealkv]] 之上原生叠加向量索引和图指针。

**选择标准**：Agent 只需要关键词 + 时间线扫描 → 纯 KV 够用。需要语义相似度检索 → 嵌入式向量索引或 SurrealDB。

## 与现有 Wiki 的关联

这与 [[agent-memory]] 的两层架构互补：
- **集成层**（框架相关）：[[fjall]]/[[slatedb]] + [[okm-framework]] 提供持久化底座
- **处理层**（框架无关）：[[graph-memory]]、向量检索等上层逻辑

## 关联页面

- [[fjall]] — 底层存储引擎
- [[composite-key-encoding]] — Key 编码基础
- [[okm-framework]] — 对象映射框架
- [[agent-memory]] — Agent 记忆两层架构
- [[graph-memory]] — 图谱化记忆
- [[raft-consensus]] — 分布式一致性
