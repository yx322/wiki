---
title: Raft 共识
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [consensus, openraft, raft, architecture]
sources: [raw/articles/kv-storage-engine.md, raw/articles/object-keyspace-mapping.md]
confidence: high
---

# Raft 共识

Raft 是一种易于理解的共识算法，提供与 Paxos 等价的强一致性保证。在分布式 KV 存储中，Raft 是状态机复制的基础。

## 核心原理

- **Leader 选举**：集群中选举一个 Leader 负责所有写操作
- **日志复制**：Leader 将写操作复制到 Follower，多数派确认后提交
- **状态机 Apply**：已提交的日志条目按顺序应用到状态机（[[fjall]]）

## 在 Aura 架构中的角色

```
[Raft 共识] → [Leader 本地状态机] → [Fjall 写入本地 NVMe] → 返回
                                         ↓
                                   WAL + SSTable
```

- [[openraft]] 提供 Raft 协议实现
- [[fjall]] 作为状态机底层存储
- 状态机将 Raft 日志条目 Apply 到 Fjall

## 为什么优于 Redlock

| 维度 | Redlock | Raft |
|------|---------|------|
| 互斥性 | 不安全（GC 停顿 + 时钟漂移） | 保证（Leader Lease + 多数派 ACK） |
| 时钟依赖 | 物理时钟（TTL） | 逻辑时钟（term + index） |
| 故障模式 | 静默丢失锁 | 显式 Leader 选举，无数据丢失 |

## 崩溃恢复

重启后从 `meta_partition` 读取 `last_log_id`，重放未 Apply 的 Raft 日志。`keyspace.persist()` 保证原子落盘。

## 何时不需要 Raft

[[slatedb]] + S3 路径下，S3 自身提供高可用和跨区域复制，Raft 的日志复制和多数派确认在 S3 之上没有增量价值。引入 Raft 只增加了运维和编码成本。

→ 详见 [[two-architecture-paths]]

## 关联页面

- [[openraft]] — Raft 实现库
- [[fjall]] — 状态机底层存储
- [[two-architecture-paths]] — 两条架构路径对比
- [[embedded-kv-vs-redis]] — Raft vs Redlock 对比
