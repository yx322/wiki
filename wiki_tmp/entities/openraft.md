---
title: Openraft
created: 2026-08-04
updated: 2026-08-04
type: entity
tags: [openraft, raft, consensus, rust]
sources: [raw/articles/kv-storage-engine.md, raw/articles/object-keyspace-mapping.md]
confidence: high
---

# Openraft

Rust 实现的 Raft 共识库，为 Fjall + Raft 架构提供分布式一致性保障。

## 核心作用

- 提供数学证明的 Raft 共识（Leader 选举 + 日志复制 + 多数派确认）
- 状态机将 Raft 日志条目 Apply 到 Fjall
- 锁状态存储在专用 Fjall 分区（`lock` CF），TTL 通过逻辑时钟检查

## 为什么优于 Redlock

| 维度 | Redlock | Openraft |
|------|---------|----------|
| 互斥性 | 不安全（GC 停顿时钟漂移） | 保证（Leader Lease + 多数派 ACK） |
| 时钟依赖 | 物理时钟（TTL） | 逻辑时钟（term + index） |
| 故障模式 | 静默丢失锁 | 显式 Leader 选举，无数据丢失 |

## 状态机集成

```rust
pub enum RaftCommand {
    UpdateActorState { agent_id: String, serialized_context: Vec<u8> },
    TerminateActor { agent_id: String },
}
```

状态机实现 `RaftStateMachine` trait 的 `apply()` 方法：
1. 遍历 Raft 日志条目
2. 匹配 `RaftCommand` 枚举
3. 写入 Fjall 物理分区
4. `keyspace.persist()` 强制刷盘

## 崩溃恢复

重启后从 `meta_partition` 读取 `last_log_id`，重放未 Apply 的 Raft 日志。

## 工程分工

| 任务 | 执行者 |
|------|--------|
| Raft 协议核心 | Openraft 库（永远不要重写共识算法） |
| 状态机 `apply` | AI + 人类审查 |
| Key 编码工具 | AI |

## 关联页面

- [[fjall]] — 状态机底层存储
- [[raft-consensus]] — Raft 共识原理
- [[state-machine-pattern]] — 状态机设计模式
