---
title: Fjall
created: 2026-07-21
updated: 2026-07-21
type: entity
tags: [fjall, embedded-kv, rust, lsm-tree, architecture]
sources: [raw/articles/kv-storage-engine.md, raw/articles/object-keyspace-mapping.md]
confidence: high
---

# Fjall

纯 Rust 实现的嵌入式 LSM-Tree KV 存储引擎（Apache-2.0），面向本地 NVMe/SSD 的极致性能场景。

## 核心特性

| 特性 | 说明 |
|------|------|
| **存储模型** | LSM-Tree，本地 NVMe/SSD |
| **异步支持** | 纯同步，需 `spawn_blocking` |
| **多空间隔离** | Partitions 物理分区（等价 RocksDB Column Family） |
| **事务** | WriteBatch 原子批量 |
| **大 Value** | WiscKey KV 分离（3.0+） |
| **时间旅行** | 不支持（需应用层 MVCC） |

## API 设计

引入 `Keyspace`（大命名空间）和 `Partition`（物理隔离分区）概念：

```rust
let keyspace = fjall::Config::new(db_path).open()?;
let user_table = keyspace.open_partition("users", PartitionCreateOptions::default())?;
let mut batch = keyspace.batch();
batch.insert(&user_table, b"key", b"value");
batch.commit()?;
```

## 在 Aura 架构中的角色

作为双引擎模式的主力：Fjall + [[openraft]]（Raft 共识），真理源在本地 NVMe。

- **写入延迟**：μs 级，不受网络波动影响
- **容量限制**：受本地磁盘约束（可通过 JuiceFS 卸载到 S3）
- **ACID 保证**：WriteBatch 在 WAL 中一次原子提交，崩溃时整体回滚

## 适用场景

- API 网关、高并发中间件
- 本地 bare-metal 极致延迟
- 需要分布式锁（配合 Openraft）
- 私有化部署（不依赖云厂商）

## 与竞品对比

| 维度 | [[fjall]] | [[slatedb]] | [[surrealkv]] |
|------|-----------|-------------|---------------|
| 真理源 | 本地 NVMe | S3 云存储 | 本地磁盘 |
| 异步 | ❌ 纯同步 | ✅ 纯异步 | ❌ 纯同步 |
| 多空间隔离 | ✅ Partitions | ❌ 扁平 | ❌ 扁平 |
| 事务 | WriteBatch | 基础批量 | ✅ 严格 MVCC |
| 时间旅行 | ❌ | ❌ | ✅ |

→ 详见 [[embedded-kv-vs-redis]]

## 关联页面

- [[slatedb]] — 云原生替代方案
- [[surrealkv]] — MVCC 事务替代方案
- [[composite-key-encoding]] — Fjall 上的数据编码方式
- [[raft-consensus]] — 分布式共识层
- [[wisckey-separation]] — 大 Value 场景的键值分离
- [[application-mvcc]] — 应用层 MVCC 实现
