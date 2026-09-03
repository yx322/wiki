---
title: SlateDB
created: 2026-08-04
updated: 2026-08-04
type: entity
tags: [slatedb, embedded-kv, rust, cloud-native]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# SlateDB

纯 Rust 云原生 LSM-Tree KV 引擎（Apache-2.0），真理源在 S3 对象存储。

## 核心特性

| 特性 | 说明 |
|------|------|
| 存储模型 | LSM-Tree，MemTable → S3 |
| 异步 | 纯异步 `.await`，融合 Tokio |
| 多空间隔离 | 扁平键空间 |
| 事务 | 基础批量原子写 |
| 点查延迟 | ms 级（S3 Range Get 网络往返） |
| 容量 | 无限（S3 桶） |

## API 特点

所有 API 天生异步，初始化直接绑定网络对象桶：

```rust
let object_store = AmazonS3Builder::from_env().build()?;
let db = slatedb::Db::open_with_opts(path, DbOptions::default(), Arc::new(object_store)).await?;
db.put(b"key", b"value").await?;
```

## 架构路径：SlateDB + S3

- 真理源在 S3，S3 提供 11 个 9 的可靠性
- 计算节点无状态，崩溃后新机器挂载同一 S3 路径秒级复活
- Scale-to-Zero 的物理基础
- **Raft 在此路径下冗余** — S3 已提供高可用

## 适用场景

- Serverless AI Agent、云原生知识库
- 缩容至零
- 无限存储需求

## 关联页面

- [[fjall]] — 本地磁盘替代方案
- [[cloud-native-storage]] — 云原生架构模式
- [[raft-consensus]] — 为何 SlateDB 路径不需要 Raft
