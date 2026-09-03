---
title: Fjall
created: 2026-08-04
updated: 2026-08-04
type: entity
tags: [fjall, embedded-kv, rust, lsm-tree]
sources: [raw/articles/kv-storage-engine.md, raw/articles/object-keyspace-mapping.md]
confidence: high
---

# Fjall

纯 Rust 实现的嵌入式 LSM-Tree KV 存储引擎（Apache-2.0）。

## 核心特性

| 特性 | 说明 |
|------|------|
| 存储模型 | LSM-Tree，本地 NVMe/SSD |
| 异步 | 纯同步，需 `spawn_blocking` |
| 多空间隔离 | Partitions 物理分区（等价 RocksDB Column Family） |
| 事务 | WriteBatch 原子批量 |
| 大 Value | WiscKey KV 分离（3.0+） |
| 时间旅行 | 不支持 |

## API 特点

引入 `Keyspace`（大命名空间）和 `Partition`（物理隔离分区）概念：

```rust
let keyspace = fjall::Config::new(db_path).open()?;
let user_table = keyspace.open_partition("users", PartitionCreateOptions::default())?;
let mut batch = keyspace.batch();
batch.insert(&user_table, b"key", b"value");
batch.commit()?;
```

## 适用场景

- API 网关、高并发中间件
- 本地 bare-metal 极致延迟
- 需要分布式锁（配合 Openraft）
- 私有化部署（不依赖云厂商）

## 在 Aura 架构中的角色

作为双引擎模式的主力：Fjall + Openraft（Raft 共识），真理源在本地 NVMe。
写入延迟 μs 级，不受网络波动影响。

## 关联页面

- [[slatedb]] — 云原生替代方案
- [[surrealkv]] — MVCC 事务替代方案
- [[composite-key-encoding]] — Fjall 上的数据编码方式
- [[raft-consensus]] — 分布式共识层
