---
title: Agent 记忆架构
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [agent, memory, architecture]
sources: [raw/articles/agent-memory-architecture.md, raw/articles/agent-memory-architecture-summary.md]
confidence: high
---

# Agent 记忆架构

两层架构：集成层（框架相关）和处理层（框架无关）。

## 两层分离

```
┌─────────────────────────────────┐
│         集成层（框架相关）         │
│  触发时机 │ 注入方式 │ 框架适配   │
├─────────────────────────────────┤
│         处理层（框架无关）         │
│  提取     │ 存储     │ 检索      │
└─────────────────────────────────┘
```

| 层 | 职责 | 演进时改什么 |
|----|------|------------|
| **集成层** | 什么时候捕获、怎么注入 prompt、怎么适配框架 | 换框架时改 |
| **处理层** | 对话→记忆的转换、存储格式、检索算法 | 换存储/检索策略时改 |

## 四个核心概念

### Session Message（会话消息）
保存真实聊天记录。不做加工，短期使用。类似人的**短期记忆**。

### Checkpoint（检查点）
对大量聊天进行阶段总结。解决历史聊天太长、Token 太多的问题。

```
msg1..msg100  →  checkpoint1（压缩摘要）
msg101..msg200 →  checkpoint2（基于 checkpoint1 + 新增消息）
```

类似 Git commit，增量式而非每次重新提交全部。

### Memory（长期记忆）
保存重要信息（用户偏好、项目背景、技术选型等），跨会话持久化。

### Cache（缓存）
不是 Memory。在 LLM 服务端或推理框架中，复用已计算的 KV Cache。详见 [[kv-cache]]。

## 集成层设计

### 触发时机

| 方式 | 机制 | 优点 | 缺点 |
|------|------|------|------|
| auto hooks | 每轮系统自动 | 零侵入 | 捕获冗余 |
| 手动调用 | Agent 判断 | 精准 | 可能遗漏 |
| 阈值触发 | N 条时 LLM 批量提取 | 低频调用 | N 条内未压缩 |

### 注入方式

| 方式 | cache 友好度 |
|------|-------------|
| 全量注入 | 差（内容每轮变化） |
| top-K 检索 | 中（检索结果可能变化） |
| **checkpoint + 增量** | **高（checkpoint 固定，可永久缓存）** |

### Checkpoint + 增量机制

类似 event-sourcing 的快照模式：

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← 前面的走 KV cache
Turn N: 达到阈值 → 压缩为 ckpt_1 → 重置
Turn N+1: [ckpt_1] + [msg_201]               ← 新 checkpoint
```

**压缩时的缓存利用**：阈值到达时，当前 prompt 已全部在 KV cache 中。此时在末尾追加压缩指令，LLM 以缓存的完整上下文处理，成本极低。指令是临时的——输出保存后丢弃，不写入 session。

## 处理层设计

### 提取格式

| 格式 | 结构 | 适用场景 |
|------|------|---------|
| flat KV | content + category + importance + tags | 简单偏好/事实 |
| graph triples | subject + predicate + object + category | 有关系结构的知识 |

### 检索

| 算法 | 机制 | 确定性 |
|------|------|--------|
| 向量（HNSW） | 语义相似度 | 概率性 |
| BM25 | 关键词匹配 | 确定性 |
| 图遍历 | 实体关系链 | 确定性 |
| RRF 融合 | 多路排序融合 | — |

## 计算时机光谱

```
写时重 ─────────────────────────────────── 读时重

LLM Wiki        KG (属性图)       KV 模式         向量模式
                graph-memory      skillforge      agentmemory

写: LLM 整理/嵌入/拓扑推断    写: 简单切分/存储
读: 直接载入/遍历              读: 匹配/排序/向量搜索
```

## 参见
- [[graph-memory]] — 图谱化记忆
- [[kv-cache]] — KV Cache 机制
- [[agent-compound-interest]] — Agent 复利

^[raw/articles/agent-memory-architecture.md]
^[raw/articles/agent-memory-architecture-summary.md]
