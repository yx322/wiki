---
title: KV 复合键编码高级技术
created: 2026-08-12
updated: 2026-08-12
type: concept
tags: [kv-storage, composite-key, encoding, fjall]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# KV 复合键编码高级技术

在纯 KV 引擎上实现复杂数据结构的关键技术。KV 只有 `put/get/delete/scan` 四个原语，一切数据结构都通过 Key 空间编码在字节序上模拟。

## 大端序补零的工程陷阱

**绝对不能**将数字转为字符串后拼入 Key。字典序与数值序不同：

```
字符串序:  "9" > "10"   (比较 '9' > '1')
数值序:    9  < 10
```

正确做法：固定宽度大端序字节数组。`i64::to_be_bytes()` 保证数值大小与字节字典序严格一致。

## 补码反转编码（Complement Encoding）

实现"最新数据排在最上面"的物理黑客手段：

```rust
// 标准正序：老数据在上面，新数据在下面
let key_asc = format!("log:{}:{}:", session_id, timestamp);

// 倒序：最新写入天然排在最前面
let inverted_time = u64::MAX - timestamp;
// 时间戳越大（越新）→ 减出来越小 → 字典序越靠前
```

**应用场景**：Agent 对话历史倒序加载、排行榜取最新记录、日志时间线倒序读取。

## 双写原子性

更新 ZSET 分数涉及两步：删除旧分数 Key + 写入新分数 Key。必须在同一个原子批次内完成：

```rust
let mut batch = keyspace.batch();
batch.delete(old_score_key);    // 旧分数索引
batch.put(new_score_key, b""); // 新分数索引
batch.put(data_key, &updated);  // 数据主表
keyspace.write(batch)?;         // 原子写入
```

不使用原子批次 → 崩溃导致只写了一半 → 索引产生脏数据。

## 二级索引末尾必须追加主键 ID

高并发场景下，两个操作可能在同一微秒产生完全相同的时间戳。末尾追加 `{entity_id}` 保证 Key 绝对唯一：

```
# 错误：同时间戳覆盖
idx:time:1722500000 → msg_abc  (被覆盖)

# 正确：主键 ID 保证唯一
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

## 哈希前缀打散

自增序列（高频事件流水号）集中撞击 LSM-Tree 末尾（Hotspot），compaction 产生严重写放大。

解法：Key 最前端注入哈希盐值，将连续写入打散到多个分区：

```rust
let hash_prefix = (murmur3_32(&key) % 16) as u8;
// sharded_key = [hash_prefix] + "metrics:ts:" + timestamp
```

**代价**：前缀扫描需要遍历所有桶。适合写密集、读按精确 Key 点查的场景。

## 长度前缀编码

消除低效的字符串分割扫描：

| 维度 | 冒号分隔 `a:b:c` | 长度前缀 `[2]a[1]b[1]c` |
|------|-----------------|------------------------|
| 反序列化 | 循环 split（O(N)）| 2 字节读长度+指针偏移（O(1)）|
| 二进制安全性 | 字段内不能含 `:` | 任意字节均可 |
| 固定宽度 | 否 | 是 |
| 适用场景 | 人类可读调试 | 生产环境高频读写 |

## 多租户复合键编解码器（工业级）

零堆分配的序列化/反序列化：

```rust
pub struct AgentMemoryKey {
    pub tenant_id: u32,       // 4 字节（租户隔离）
    pub session_id: [u8; 16], // 16 字节（UUID 原始二进制）
    pub timestamp: u64,       // 8 字节（倒序时间戳）
}
// 总计 30 字节，反序列化纯指针切片操作，纳秒级
```

## SQL 操作的 KV 实现

### 倒排索引交集（多维查询）

```sql
SELECT * FROM orders WHERE status='shipped' AND region='east';
```

KV 实现：
1. **写入**：每个维度独立建索引，原子 Batch 同时提交
2. **读取**：两次前缀扫描 → 归并交集（双指针 O(N+M)）→ 回表点查

**排序零成本是 KV 的结构性优势**：LSM-Tree 的字典序就是排序，省掉 SQL 的 O(N log N) 排序步骤。

### 去范式化 vs 应用层 JOIN

| 策略 | 适用场景 | 读 | 写 |
|------|---------|-----|-----|
| **写时去范式化** | 读密集 | 一次 get 拿全部 | 写入时冗余嵌入 |
| **读时 JOIN** | 写密集 | 多次 get + 内存合并 | 各实体独立写入 |

## 参见
- [[design-patterns]] — 纯 KV 四大设计模式
- [[composite-key-encoding]] — 复合键编码基础
- [[fjall]] — Fjall 引擎
- [[sql-vs-kv-pipeline]] — SQL vs KV 管道链

^[raw/articles/kv-storage-engine.md]
