---
title: SurrealKV
created: 2026-07-21
updated: 2026-07-21
type: entity
tags: [surrealkv, embedded-kv, rust, lsm-tree, architecture]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# SurrealKV

纯 Rust 嵌入式 KV 引擎，以 ACID 事务为核心设计理念。SurrealDB 的底层存储内核。

## 核心特性

| 特性 | 说明 |
|------|------|
| **存储模型** | LSM-Tree，本地磁盘 |
| **异步支持** | 纯同步 |
| **多空间隔离** | 扁平键空间 |
| **事务** | 严格 MVCC 事务 |
| **大 Value** | Blob Log 大对象分离 |
| **时间旅行** | Versioned Queries（内置 `tx.get_at(key, timestamp)`） |

## API 设计

所有读写必须包裹在 Transaction 闭包中：

```rust
let kv = surrealkv::Store::new(Options::new(db_path))?;
let mut tx = kv.begin_rw()?;
tx.set(b"key", b"value")?;
tx.commit()?;
```

## 与 Fjall 应用层 MVCC 的区别

SurrealKV 内置 `tx.get_at(key, timestamp)` 直接查询历史版本，不需要应用层将版本号编入 Key。
[[fjall]] 需要手动编码版本号（`data:entity:v:[u64::MAX - version]`）。代价不同，效果相同。

→ 详见 [[application-mvcc]]

## 适用场景

- 并发账务、历史版本回滚
- 需要原生时间旅行的场景
- 省去千行应用层版本维护代码

## 关联页面

- [[fjall]] — 性能优先替代方案
- [[slatedb]] — 云原生替代方案
- [[application-mvcc]] — Fjall 上的应用层 MVCC
- [[surrealdb]] — SurrealKV 的上层数据库
