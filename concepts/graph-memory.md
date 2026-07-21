---
title: 图谱化记忆 (Graph Memory)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [agent, memory, graph, surrealdb]
sources: [raw/articles/graph-memory.md]
confidence: high
---

# 图谱化记忆 (Graph Memory)

记忆架构的演进方向：提取从 flat KV 升级为 graph triples，检索增加图遍历。

## 核心思路

存储原子化（精确、无冗余），检索聚簇化（连贯、有语境）。

### 文档 vs 原子三元组

| 维度 | 文档 | 原子三元组 |
|------|------|----------|
| 语义稳定性 | 漂移（修改改变语义位置） | 不变（新知识=新边） |
| 信息密度 | 低（大量噪声） | 高（1 fact = 1 triple） |
| 查询方式 | 关键词/向量（概率性） | 图遍历（确定性） |
| 更新复杂度 | O(n²)（原地修改需读全文） | O(n)（只追加不读旧） |

## 双层模型

存储要原子化，检索要上下文化。同一个粒度无法同时满足两个需求。

```
写入层（原子）: 一条边 = 一个事实，只追加
↓
组织层（聚簇）: 相关边按实体/事件/主题分组
↓
检索层（视图）: 以簇为单位返回，不返回散装边
```

## 三种聚簇策略

### 1. 实体锚定
所有提及实体 X 的边，天然挂在 X 节点上。检索时找实体→返回邻域子图。

### 2. 时间线聚合
同一事件/决策的所有边，按时间窗口分组。解决"某个决策当时考虑了什么"。

### 3. 因果链聚合
按"为什么"的推理链分组。不是按实体（谁）、不是按时间（何时），而是按因果逻辑（为什么）。

## 两阶段提取管线

直接从口语/多人对话抽取三元组，信号会被噪声淹没。

### 阶段 1：规范化（保义，不损义）
- ✅ 指代消解（他→张三）
- ✅ 去重合并（三遍同一观点→一条）
- ✅ 归属标注（A说的 vs B说的 vs 共识）
- ✅ 口语→书面
- ❌ 不做总结、不改写语义

### 阶段 2：分类提取

| 类别 | 三元组模式 | 例子 |
|------|----------|------|
| 事实 | (entity, relation, entity) | (Postgres, supports, pgvector) |
| 规律 | (condition, implies, result) | (高并发+写密集, implies, 用PG不用SQLite) |
| 逻辑 | (premise, therefore, conclusion) | (cognee不支持SurrealDB, therefore, 选Postgres) |
| 偏好 | (user, prefers, thing) | (user, prefers, 保守操作) |

## 权重系统

每条边携带系统级元数据：

```
weight = read_score + write_score + tool_score + rarity_bonus - time_decay

read_score  = log(1 + read_count)
write_score = log(1 + write_count) × 0.5
tool_score  = log(1 + tool_invoke_count) × 1.5  # 工具调用权重最高
```

### 读写比矩阵

| 象限 | read | write | 含义 | 处理 |
|------|------|-------|------|------|
| 核心区 | 多 | 多 | 反复使用、反复确认 | 最高权重 |
| 深水区 | 多 | 少 | 有人查但门槛高 | 高权重+稀缺性加成 |
| 噪声区 | 少 | 多 | 写入频繁但无人查询 | 降权 |
| 冷门区 | 少 | 少 | 低价值 | 降级为冷数据 |

## 涌现式 Skill 边界

工具指数高的边自然聚集成"频繁使用的操作链"——这就是 Skill 的隐式定义。Skill 不是人工划定的边界，是使用数据涌现出来的子图。

## 参见
- [[agent-memory]] — 记忆架构总览
- [[surrealdb]] — 数据库实现
- [[hybrid-search]] — 混合搜索（向量+BM25+RRF）

^[raw/articles/graph-memory.md]
