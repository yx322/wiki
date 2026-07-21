---
title: 应用层 MVCC
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, lsm-tree, architecture, performance]
sources: [raw/articles/kv-storage-engine.md, raw/articles/object-keyspace-mapping.md]
confidence: high
---

# 应用层 MVCC

[[fjall]]/[[slatedb]] 不支持原生 MVCC。通过将版本号编排进 Key 骨架，实现应用层多版本控制。

## 实现原理

版本号编入 Key，配合补码反转实现"最新优先"：

```
Key: data:agent:101:v:[u64::MAX - 1001]  →  记忆状态 v1001
Key: data:agent:101:v:[u64::MAX - 1002]  →  记忆状态 v1002（最新）
```

- **常规读取**：`Scan("data:agent:101:v:").next()` → 补码反转后最新版本排最前，亚微秒拿到最新状态
- **时间旅行**：将前缀指针定位到目标版本号之后，正向扫描即得历史版本链
- **无需数据库快照锁**，纯 Key 设计实现无锁历史回滚

## 与 SurrealKV 原生 MVCC 的对比

| 维度 | 应用层 MVCC（Fjall） | 原生 MVCC（SurrealKV） |
|------|---------------------|----------------------|
| 实现方式 | 版本号编入 Key | 内置 `tx.get_at(key, timestamp)` |
| 复杂度 | 需手动编码 | 引擎内置 |
| 灵活性 | 高（自定义版本策略） | 中（固定时间戳） |
| 性能 | 相同 | 相同 |

## 版本化 Enum 懒迁移

在 OKM 框架中，通过版本化 Enum 实现零停机 Schema 演进：

```rust
pub enum MemoryValuePayload {
    V1(AgentMemoryV1),
    V2(AgentMemoryV2),  // 新增字段
}

match database.get(&key) {
    MemoryValuePayload::V1(old) => upgrade_v1_to_v2(old),
    MemoryValuePayload::V2(current) => current,
}
```

只有被读到的老记录才升级，未读到的继续以旧格式存储，不浪费写入带宽。这是 SQL `ALTER TABLE` 无法做到的——PG 的迁移必须遍历全表重写所有行。

## 关联页面

- [[fjall]] — 底层存储引擎
- [[surrealkv]] — 原生 MVCC 替代方案
- [[okm-framework]] — 版本化 Enum 实现
- [[composite-key-encoding]] — 补码反转编码
