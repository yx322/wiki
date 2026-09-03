---
title: 复合键编码（Composite Key Encoding）
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [composite-key, embedded-kv, lsm-tree, performance]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# 复合键编码

纯 KV 引擎没有 Hash/List/Set/ZSET 原语。所有数据结构都通过 Key 空间编码在字节序上模拟——LSM-Tree 迭代器天然按字节排序，只要 Key 编码设计正确，范围扫描和排序在存储层零成本完成。

## Redis 数据结构 → KV 编码映射

| Redis 类型 | KV 编码方式 | 读取 | 写入 |
|-----------|------------|------|------|
| STRING | `str:<key>` → value | get | put |
| HASH | `hash:<key>:<field>` → value | get / prefix | put / remove |
| LIST | `list:<key>:<seq>` (8位零填充) → value | range | put + monotonic seq |
| SET | `set:<key>:<member>` → "" | existence check | put / remove |
| ZSET | `zset:<key>:<score>:<member>` → "" | range (score interval) | put / remove |

## 关键技术细节

### 大端序补零（Big-Endian Zero-Padding）

**绝对不能**将数字转为字符串后拼入 Key。字典序与数值序不同：

```
字符串序: "9" > "10"  (比较 '9' > '1')
数值序:   9  < 10
```

正确做法：固定 8 字节大端序字节数组 `i64::to_be_bytes()`。

### 补码反转（Complement Encoding）

实现"最新数据排在最上面"——`u64::MAX - timestamp`：

- 时间戳越大（越新），减出来越小
- 越小 → LSM-Tree 字典序越靠前 → 物理层面"最新优先"
- 正向迭代器扫描自然得到倒序结果

应用场景：Agent 对话历史倒序加载、排行榜取最新记录、日志时间线倒序读取。

### 双写原子性

更新 ZSET 分数涉及删旧 Key + 写新 Key，必须在同一个原子 Batch 内完成。崩溃导致只写一半 → 索引产生脏数据。[[fjall]] 的 `Batch` API 保证底层 WAL 一次原子提交。

在分布式模式下，这个 Batch 作为单条 Raft 指令提交——从单机原子性延伸到集群原子性。

### 二级索引末尾必须追加主键 ID

高并发场景下，两个操作可能在同一微秒产生完全相同的时间戳。末尾追加唯一 ID 防止覆盖：

```
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

### 哈希前缀打散

自增序列导致所有写入撞击 LSM-Tree 末尾（热点）。用 murmur3 哈希取模分 16 个桶，连续递增写入被均匀分散到 16 个独立内存树，多线程并发刷盘并行处理。

### 长度前缀编码

用 2 字节长度前缀替代冒号分隔，反序列化从 O(N) 字符串扫描变为 O(1) 指针偏移。

| 维度 | 冒号分隔 `a:b:c` | 长度前缀 `[2]a[1]b[1]c` |
|------|------------------|------------------------|
| 反序列化 | 循环 split | 2 字节读长度 + 指针偏移 |
| 二进制安全性 | 字段内不能包含 `:` | 任意字节均可 |
| 固定宽度 | 否 | 是 |

## 多租户复合键（工业级实现）

零堆分配的固定宽度 Key，反序列化纯指针切片，纳秒级：

```rust
pub struct AgentMemoryKey {
    pub tenant_id: u32,        // 4 字节
    pub session_id: [u8; 16],  // 16 字节 UUID
    pub timestamp: u64,        // 8 字节倒序时间戳
}
// 总长度 = 4 + 1 + 16 + 1 + 8 = 30 字节
```

## 关联页面

- [[fjall]] — 底层存储引擎
- [[okm-framework]] — 上层对象映射
- [[wisckey-separation]] — 大 Value 场景的键值分离
- [[application-mvcc]] — 版本号编入 Key 实现 MVCC
- [[design-patterns]] — 纯 KV 底座上的四大设计模式
