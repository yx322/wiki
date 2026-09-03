---
title: Iceberg Copy-on-Write 问题
type: concept
tags: [iceberg, performance, cow, write-amplification]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/Iceberg表性能退化与架构优化分析.md
  - raw/articles/湖仓一体化性能重构落地方案.md
confidence: high
contested: false
---

# Iceberg Copy-on-Write 问题

Copy-on-Write (CoW) 模式在高频小批量更新场景下会引发严重写放大和元数据冲突。

## 问题现象

每次仅更新 17~150 条数据，`table.overwrite` 耗时高达 **39.9~45 秒**，且频繁抛出 `CommitFailedException`。

## 根源分析

### 写放大

CoW 模式下，为了主键条件不得不重写全表 190 万行文件：

```
更新 17 条数据
  ↓
命中 5 个用户 × 3 个日期 = 15 个独立分区
  ↓
同时打开 15 个目录下的旧小文件
  ↓
各自剔除、各自写入新小文件
  ↓
在元数据树上修改 15 个 Manifest 记录
  ↓
写放大：实际写入数据量远超更新数据量
```

### 乐观锁冲突

高频多线程并发修改庞大元数据树，极易触发阿里云 OSS Tables 服务端的乐观锁冲突：

```
线程 A 提交 → 修改 metadata.json 指针
线程 B 同时提交 → 发现指针已被修改 → CommitFailedException
  ↓
指数级退避重试 → 耗时拉长到 45 秒甚至超时崩溃
```

### 物理重写毫无提升

在云存储上，重写 1 个 50MB 的未分区大文件，和重写 15 个几 KB 的分区小文件，其网络请求和事务提交的固有物理开销完全相同。分区写入在这里完全没有享受到大数据的并行裁剪优势，纯属"负优化"。

## 解决方案

### 方案 1：Merge-on-Read (MoR)

写入时只追加删除标记和新数据文件，读取时合并。减少写放大，但增加读取复杂度。

### 方案 2：流数据库 + Iceberg Sink

```
数据源  →  [[risingwave]]（实时 Upsert）  →  Iceberg Sink  →  [[oss]]
           20 毫秒写入                       每 2 分钟大批次落湖
```

**关键参数**：
- `type = 'upsert'`：覆盖更新语义
- `sink.flush.interval = '120s'`：每 2 分钟大批次写入
- `sink.checkpoint.interval = '60s'`：每 1 分钟对齐元数据，物理斩断 Commit 冲突

详见 [[risingwave]]。

### 方案 3：完全不分区 + 追加写入

放弃细粒度分区，改用完全不分区表 + 追加写入 + 技术性去重视图。

详见 [[iceberg-small-file-problem]]。

## 性能对比

| 方案 | 写入延迟 | Commit 冲突 | OOM 风险 |
|------|---------|------------|---------|
| PyIceberg CoW | 45 秒 | 频繁 | 高 |
| RisingWave + Sink | 20 毫秒 | 消灭 | 无 |

## 参见

- [[iceberg]] — 表格式规范
- [[iceberg-small-file-problem]] — 小文件问题
- [[risingwave]] — 流数据库解法
