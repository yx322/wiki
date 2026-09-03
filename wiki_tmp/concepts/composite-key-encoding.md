---
title: Composite Key Encoding（复合键编码）
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [composite-key, key-encoding, lsm-tree, rust]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# Composite Key Encoding

纯 KV 引擎没有 Hash/List/Set/ZSET 原语。所有数据结构都通过 **Key 空间编码** 在字节序上模拟——LSM-Tree 迭代器天然按字节排序，只要 Key 编码设计正确，范围扫描和排序在存储层零成本完成。

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

正确做法：固定 8 字节大端序字节数组。

```rust
fn score_key(prefix: &[u8], score: i64, member: &[u8]) -> Vec<u8> {
    let mut key = prefix.to_vec();
    key.extend_from_slice(&score.to_be_bytes());  // 8 bytes, 高位在前
    key.push(b':');
    key.extend_from_slice(member);
    key
}
```

### 补码反转（Complement Encoding）

实现"最新数据排在最上面"——`u64::MAX - timestamp`：

```rust
// 标准正序：老数据在最上面，新数据在最底下
let key_ascending = format!("log:{}:{}:", session_id, timestamp).into_bytes();

// 倒序：最新写入的数据天然排在最前面
let inverted_time = u64::MAX - timestamp;
let mut key_descending = Vec::new();
key_descending.extend_from_slice(format!("log:{}:", session_id).as_bytes());
key_descending.extend_from_slice(&inverted_time.to_be_bytes());
```

**物理含义**：时间戳越大（越新），减出来越小，在 LSM-Tree 中排越靠前。正向迭代器自然得到倒序结果。

### 双写原子性

更新 ZSET 分数涉及删旧 Key + 写新 Key，必须在同一个原子 Batch 内完成：

```rust
let mut batch = keyspace.batch();
batch.delete(old_score_key);
batch.put(new_score_key, b"");
batch.put(data_key, &updated_player);
keyspace.write(batch)?;  // 原子写入
```

崩溃导致只写一半 → 索引产生脏数据。

### 二级索引末尾必须追加主键 ID

防止同时间戳覆盖：

```
# 错误：同时间戳覆盖
idx:time:1722500000 → msg_abc  (被覆盖)

# 正确：主键 ID 保证唯一
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

### 哈希前缀打散

自增序列导致所有写入撞击 LSM-Tree 末尾（热点），用 murmur3 哈希取模分 16 个桶：

```rust
let hash_prefix = (murmur3_32(&mut cursor, 0).unwrap() % 16) as u8;
sharded_key.push(hash_prefix);  // 1 字节哈希盐
```

连续递增写入被均匀分散到 16 个独立内存树，多线程并发刷盘并行处理。

### 长度前缀编码

用 2 字节长度前缀替代冒号分隔，反序列化从 O(N) 字符串扫描变为 O(1) 指针偏移：

```rust
pub fn pack_string_component(buf: &mut Vec<u8>, component: &str) {
    let len = component.as_bytes().len() as u16;
    buf.extend_from_slice(&len.to_be_bytes());
    buf.extend_from_slice(component.as_bytes());
}
```

## 多租户复合键编解码器（工业级实现）

零堆分配的固定宽度 Key：

```rust
pub struct AgentMemoryKey {
    pub tenant_id: u32,        // 4 字节
    pub session_id: [u8; 16],  // 16 字节 UUID
    pub timestamp: u64,        // 8 字节倒序时间戳
}
// 总长度 = 4 + 1 + 16 + 1 + 8 = 30 字节
```

反序列化纯指针切片，纳秒级。

## 关联页面

- [[fjall]] — 底层存储引擎
- [[okm-framework]] — 上层对象映射
- [[wisckey-separation]] — 大 Value 场景的键值分离
- [[application-mvcc]] — 版本号编入 Key 实现 MVCC
