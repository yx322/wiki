---
title: SQL vs KV 管道链
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, comparison, architecture, performance]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# SQL vs KV 管道链

对于框架平台（API 网关、Agent 执行器），数据访问路径在设计期就已固化。去掉 SQL 不是为了省事，而是消灭数据库优化器（Query Planner）这个运行时黑盒，获得 100% 的物理性能确定性。

## 实际对比：拉取最近 10 条对话记忆

### SQL 方式（PostgreSQL / SurrealDB）

```sql
SELECT message_id, content FROM agent_memories
WHERE session_id = 'session_456'
ORDER BY timestamp DESC LIMIT 10;
```

底层物理代价：词法语法分析 → AST 树生成 → 逻辑执行计划 → 优化器猜测索引扫描还是全表扫描 → B-Tree 节点间频繁跳转。

### KV 方式（Rust + 嵌入式 KV）

写入时 Key 已编排为倒序物理格式：`m:{session_id}:{u64_max - timestamp}:{message_id}`。

```rust
let prefix = format!("m:session_456:").into_bytes();
let top_10 = kv_engine
    .scan(&prefix)         // 定位起始区间（O(log N) 内存二分）
    .take(10)              // 顺序读 10 条（O(1) 磁盘/内存顺序 I/O）
    .collect::<Vec<_>>();
```

## 为什么 KV 胜出

- Rust 方法链比 SQL 声明式样板更精炼
- 没有黑盒优化器自作聪明——代码就是执行路径
- 数据从 10MB 到 10TB，执行效率不变，亚毫秒响应雷打不动

## 生产级组件：原子双写 + 时间线索引

用一个原子 Batch 在写入主数据的同时，自动构建时间线倒序二级索引：

```rust
pub fn save_agent_memory(&self, session_id, message_id, timestamp, payload) -> TransactionBatch {
    let mut batch = TransactionBatch { actions: Vec::new() };

    // 1. 主表：完整数据
    let data_key = format!("data:session:{}:msg:{}", session_id, message_id).into_bytes();
    batch.put(&data_key, payload);

    // 2. 时间线索引：倒序排列（u64::MAX - timestamp）
    let inverted_time = u64::MAX - timestamp;
    let mut index_key = Vec::new();
    index_key.extend_from_slice(format!("idx:time:session:{}:", session_id).as_bytes());
    index_key.extend_from_slice(&inverted_time.to_be_bytes());
    index_key.extend_from_slice(format!(":{}", message_id).as_bytes());
    batch.put(&index_key, &[]);  // Value 为空——实体 ID 已编码在 Key 中

    batch
}
```

主表存完整数据，索引表只存空 Value。一次原子提交，两个 Key 同时成功或同时失败。SQL 的 `INSERT INTO` 无法在一个语句中同时写入两张表并保证原子性。

## KV 框架 vs 自研数据库

纯 KV 之上叠加 Parser + Optimizer 就是一个完整的数据库引擎：

| 数据库 | 查询层 | 存储内核 | 本质 |
|--------|--------|---------|------|
| TiDB | MySQL 语法解析器 | TiKV（Rust KV） | SQL 翻译器 + KV |
| CockroachDB | PostgreSQL 语法兼容 | Pebble（Go KV） | SQL 翻译器 + KV |
| SurrealDB | SurrealQL 函数式解析器 | SurrealKV（Rust KV） | DSL 翻译器 + KV |

它们没有发明新的磁盘驱动器，只是在 KV 之上盖了一层解析器和优化器外壳。

## 何时必须蜕变为数据库

只有当系统需要开放给外部第三方开发者、允许最终用户通过低代码/动态插件自由写出不可预测的复杂查询时，才必须在最前面加一层 Parser + Optimizer 做查询门禁。

**判定**：框架平台的正确姿态是坚守纯 KV + [[composite-key-encoding]]。数据库是 KV 的上层封装，不是 KV 的替代。

## SQLite vs 嵌入式 KV

| 维度 | SQLite | 嵌入式 KV（Fjall） |
|------|--------|-------------------|
| C 语言依赖 | 需要 gcc/clang | 纯 Rust，零外部依赖 |
| 双重缓存 | Page Cache + 应用层 | 直接映射，更短读取路径 |
| 写锁 | 数据库级排他锁，`SQLITE_BUSY` | 无锁 MemTable，多核并行 |

现实案例：Docker 用 bbolt（Go KV），K3s 从 SQLite 向 etcd 嵌入式 KV 收敛。

## 关联页面

- [[composite-key-encoding]] — Key 编码基础
- [[design-patterns]] — 纯 KV 四大设计模式
- [[okm-framework]] — 代码即 DDL 的实现
- [[embedded-kv-vs-redis]] — 与 Redis 的对比
