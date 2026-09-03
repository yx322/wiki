---
title: 两条架构路径
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, architecture, fjall, slatedb, consensus]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# 两条架构路径：Fjall + Raft vs SlateDB + S3

[[fjall]] 和 [[slatedb]] 都是纯 Rust LSM-Tree KV 引擎（Apache-2.0），底层数学逻辑相似。但它们在真理源（Source of Truth）和网络拓扑上走向了相反的极端。

## 引擎定位对比

| 维度 | Fjall | SlateDB |
|------|-------|---------|
| **真理源** | 本地 NVMe/SSD | 云端对象存储（S3/GCS/MinIO） |
| **Flush 路径** | MemTable → 本地磁盘 | MemTable → S3 |
| **点查延迟** | μs 级（本地 NVMe） | ms 级（S3 Range Get） |
| **容量上限** | 本地磁盘 | 无限（S3 桶） |
| **设计目标** | 单机 bare-metal，极致延迟 | 云原生，节点无状态化 |

## 路径一：Fjall + Raft

```
[Raft 共识] → [Leader 本地状态机] → [Fjall 写入本地 NVMe] → 返回
```

- **真理源在本地磁盘**
- Raft 达成共识后，状态机无网络损耗地写入本地 Fjall
- 延迟由 NVMe 物理特性决定（μs 级），不受网络波动影响
- [[openraft]] 提供 Raft 共识保障

## 路径二：SlateDB + S3

```
[gRPC 计算节点（无状态）] → [SlateDB] → [S3 桶] → 返回
```

- **真理源在 S3**
- S3 本身提供 11 个 9 的可靠性和跨区域复制
- 计算节点无状态，崩溃后新机器挂载同一 S3 路径秒级复活
- **Raft 在此路径下冗余**——S3 已提供高可用

## 选择标准

| 指标 | Fjall + Raft | SlateDB + S3 |
|------|-------------|--------------|
| **写延迟极限** | < 1ms（亚毫秒） | 可接受 1-10ms |
| **私有化部署** | 必须（不依赖云厂商） | 可选（S3 是唯一外部依赖） |
| **运维模型** | 自管磁盘/Raft 集群 | 云厂商管存储，自管无状态计算 |
| **容量** | 本地磁盘上限（可 JuiceFS 卸载） | 无限（S3 桶） |

## 判定

- **Fjall + Raft**：对延迟敏感、需要完全私有化部署（如 AI Agent 记忆库、实时网关）
- **SlateDB + S3**：云原生 Serverless 架构（无状态计算 + 无限存储），代价是延迟上限更高且依赖云厂商

## 关联页面

- [[fjall]] — 路径一核心引擎
- [[slatedb]] — 路径二核心引擎
- [[openraft]] — 路径一共识层
- [[raft-consensus]] — Raft 共识原理
- [[embedded-kv-vs-redis]] — 与 Redis 的对比
