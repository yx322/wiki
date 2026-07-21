---
title: 存储进化链：红黑树 → B-Tree → LSM-Tree → 开放表格式
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [architecture, performance, lsm-tree, lakehouse, embedded-kv]
sources: [raw/articles/storage-evolution-chain.md]
confidence: high
---

# 存储进化链：红黑树 → B-Tree → LSM-Tree → 开放表格式

从微观到宏观的完整谱系：

```
红黑树（内存，单元素节点）
   ↓ 节点变大、压树高
B-Tree（磁盘，大节点）
   ↓ 写优化、顺序追加
LSM-Tree（磁盘，多文件 + manifest）
   ↓ 文件变大、分布式、对象存储
开放表格式（S3/OSS，Parquet + 元数据树）
```

## 每一级跃迁的核心驱动力

### 1. 红黑树（内存，单元素节点）
- **跃迁驱动**：追求内存中的绝对平衡与极速查找。

### 2. B-Tree（磁盘，大节点）
- **跃迁驱动**：为了对抗磁盘 IO 的龟速。既然去一次仓库太费劲，就把节点变大（打包塞满数据），把树压矮，争取一次 IO 拿回一大批数据。

### 3. LSM-Tree（磁盘，多文件 + manifest）
- **跃迁驱动**：B-Tree 虽然读得快，但写的时候到处找位置（随机写）太慢了。于是 LSM-Tree 放弃原地修改，改为"先写内存，再顺序追加到磁盘（顺序写）"。代价是读的时候可能需要查多个文件，所以引入了后台的合并（Compaction）和清单（Manifest）来管理这些文件。

### 4. 开放表格式（S3/OSS，Parquet + 元数据树）
- **跃迁驱动**：随着数据量达到 PB 级，单机磁盘装不下了，必须走向分布式和云原生（对象存储 S3/OSS）。这时候，数据文件变成了 Parquet/ORC 这种适合列式分析的格式，而原本 LSM-Tree 里的 Manifest 思想被放大，变成了强大的"元数据树（Metadata Tree）"（比如 [[iceberg]] 的 Metadata/Manifest List/Manifest File 三级结构），用来在分布式环境下管理海量文件、实现时间旅行和 ACID 事务。

## 总结：存储进化史的本质

从红黑树 → B-Tree → LSM-Tree → 开放表格式，这条进化链的本质，就是人类在"计算速度 vs 存储介质速度 vs 数据规模"这三者之间，不断寻找最优解的过程：

1. **内存时代**：怎么快怎么来（红黑树）
2. **单机磁盘时代**：为了少跑几趟磁盘，把树压扁（B-Tree）
3. **高并发写入时代**：为了写得快，把随机写变成顺序追加（LSM-Tree）
4. **云原生大数据时代**：为了存得下、算得快、多引擎共享，把结构打散到云端，用元数据树来统领全局（开放表格式）

## 微观与宏观的联系

| 层级 | 问题 | 解法 |
|------|------|------|
| **微观**（单库内部） | 10 亿行数据在一块磁盘上，怎么 3 次 IO 找到一行？ | B-Tree（节点打包、压树高） |
| **宏观**（分布式存储） | 10 亿行数据散在 OSS 上 10000 个 Parquet 文件里，怎么找到对的那几个文件？ | 开放表格式（元数据索引 + 文件级裁剪） |

[[iceberg]] 的 manifest 树其实就是"文件级 B-Tree"——用树状元数据把搜索空间层层缩小，只是节点里存的不是行数据，而是文件指针和统计信息。

## 参见

- [[btree-vs-redblack-tree]] — 红黑树 vs B-Tree 的微观对比
- [[fjall]] — LSM-Tree 引擎实现
- [[iceberg]] — 开放表格式代表
- [[delta-lake]] — 开放表格式代表
- [[lakehouse]] — 湖仓一体架构
- [[sql-vs-kv-pipeline]] — B-Tree（SQL）vs LSM-Tree（KV）的性能对比

^[raw/articles/storage-evolution-chain.md]
