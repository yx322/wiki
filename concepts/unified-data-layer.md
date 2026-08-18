---
title: 统一数据层架构
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [surrealdb, architecture, polyglot-persistence, compute-pushdown]
sources: [raw/articles/unified-data-layer.md]
confidence: high
---

# 统一数据层架构

SurrealDB 作为统一数据层的架构哲学——用单一多模型引擎替代传统的"多语种持久化"（Polyglot Persistence）。

## 核心哲学

传统的 KV + 图 + 关系型多数据库拼凑架构会制造运维复杂性和数据碎片化。SurrealDB 试图将这些模型统一到一个引擎中。

## SQL 的根本性缺陷

### 语言缺陷
- **语法/执行顺序相悖**：书写 SELECT 在前，执行 FROM 在前
- **三值逻辑（NULL）**：`NOT IN` 子查询含 NULL 时返回空结果

### 架构缺陷：缺乏组合性
- **数据搬运**：无法在查询内组合逻辑 → 被迫拉到应用层编排 → 不必要的网络往返
- **ORM 是数据领域的"Go 语言"**：通过阉割底层表达力换取虚假舒适感

## SurQL 六大架构级解决方案

| 方案 | 说明 |
|------|------|
| **计算下推** | 在数据库层直接编写复杂业务逻辑 |
| **原生可组合性** | 查询是**值**（`LET $x = SELECT ...`），不是文本 |
| **多模型统一** | `->knows->` 图遍历免 JOIN |
| **原生实时推送** | `LIVE SELECT` WebSocket 订阅 |
| **反 ORM** | 分布式可扩展 + 原生执行 |
| **现代语法** | 泛 Rust 血统（let/管道/类型后置/表达式）|

## 计算图参照系

| 维度 | SQL | Polars/Spark | SurQL |
|------|-----|-----------|-------|
| 载体 | 字符串 | 方法链计算图 | 一等公民值 |
| 组合性 | 无 | 有 | 有 |
| 控制流 | 无 | 无 | 有（图灵完备）|
| 执行边界 | DB 内（被迫搬到应用层）| 进程内 | DB 内（计算下推）|

## 关系建模对比

### 一对多：三种机制递进
1. **Record ID 指针** — 简单一对多
2. **图的边（RELATE）** — 边可带属性
3. **多态引用** — 一个字段指向任意表

### 多对多：中间表的消亡

| 方案 | 物理结构 | 关系属性 | 查询复杂度 |
|------|---------|---------|-----------|
| SQL | 3 张表 | 中间表加列 | 多次 JOIN |
| SurrealDB 图模式 | 2 点+1 边 | RELATE SET | 箭头穿透一行代码 |
| SurrealDB 数组模式 | 2 张表 | 不支持 | `.*` 展开 |

**⚠️ 僵尸指针**：数组模式下删除实体后引用不自动消失。

## SurrealDB vs PostgreSQL

| 层次 | SurrealDB 碾压 | PG 碾压 |
|------|---------------|--------|
| 范式 | 语言设计、多态关联、逻辑下沉、RTT | — |
| 生态 | — | 成熟度、数据一致性、工具链、云厂商 |

**判定**：四个范式维度 SurrealDB 碾压，四个生态维度 PG 碾压。互有胜负，没有全面碾压。

## Redis 替代论证

- **Redis 神话**："Redis 快因为它是内存数据库"
- **现实**：网络 RTT 主导延迟，无论后端是 Redis 还是索引良好的 DB
- **权衡**：相同网络延迟成本下，你获得丰富的关系（图）+ 结构化数据 + 实时订阅，对比 Redis 只是 KV

## OLTP vs OLAP 边界

- **OLTP**：SurrealDB 通过减少网络往返表现出色
- **OLAP**：最终答案是 Lakehouse（Delta Lake + S3 + Parquet），SurrealDB 通过 CDC 将历史数据传输到湖中

## Embedding 计算下推案例

| 路径 | 传输次数 | 存储位置 |
|------|---------|---------|
| Python 侧生成 | Ollama→Python + Python→SurrealDB = **2 次** | **2 处** |
| SurrealQL 内部生成 | Ollama→SurrealDB = **1 次** | **1 处** |

```surql
CREATE memories SET
    content = $content,
    embedding = (fn::ollama::embed('bge-m3', $content)).embeddings[0];
```

## 挑战者逻辑

**技术颠覆铁律**：挑战者必须在足够多维度上达到"压倒性优势"才能克服生态惯性。七八个持平或略好 + 一两个强很多 + 一个稍弱但不太差 = 挑战成功。半斤八两 = 挑战失败。

**扩展性是次生问题**：现代数据库走存算分离路线。想解决根本性问题，SQL 是绕不过去的。

## 参见
- [[surrealdb]] — SurrealDB 实体
- [[query-language-design]] — 查询语言设计（本文的精简版）
- [[modern-language-design]] — SurQL 的泛 Rust 血统
- [[hybrid-search]] — 混合搜索（SurQL 的实际应用）
- [[embedded-kv-vs-redis]] — 嵌入式 KV vs Redis

^[raw/articles/unified-data-layer.md]
