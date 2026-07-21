---
title: Iceberg 小文件问题
type: concept
tags: [iceberg, performance, small-file-problem, oss]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/iceberg-storage-architecture-and-optimization.md
  - raw/articles/Iceberg表性能退化与架构优化分析.md
confidence: high
contested: false
---

# Iceberg 小文件问题

在小数据量 + 过度分区场景下，Iceberg 会严重退化。

## 问题现象

190 万行（~50MB）的轻量级数据：
- **查询加载**：[[pyiceberg]] 加载到内存耗时 **19.6 秒**
- **高频更新写入**：每次仅 17~150 条数据，`table.overwrite` 耗时 **39.9~45 秒**
- **频繁报错**：`CommitFailedException` 元数据提交冲突，引发指数级重试

## 根源诊断

### 高维稀疏状态

当表配置为 `500 用户 × 365 天` 的二级分区时：

| 指标 | 数值 |
|------|------|
| 物理目录爆炸 | 18 万 ~ 36.5 万个物理目录 |
| 数据严重稀疏 | 190 万行 ÷ 36.5 万分区 ≈ **5.2 行/分区** |

每个分区目录平均只躺着 5 条数据。数据本身（Parquet）仅 50MB，但描述 36.5 万个分区的元数据（`metadata.json` 和 Manifest 文件）膨胀到几个 GB。

## 读取流瓶颈

[[duckdb]] 查询是毫秒级，但必须等待 [[pyiceberg]] 构建好 PyArrow Table。19.6 秒被拦截在前端：

- **元数据解析卡死**：PyIceberg 单线程从 OSS 下载并解析包含 36.5 万个分区路径的庞大 `metadata.json`，光解析就卡住 15 秒以上
- 分区裁剪省下的数据下载时间，在恐怖的元数据解析开销面前毫无意义

## 写入流瓶颈

Copy-on-Write (CoW) 模式下的 Sync-on-Query 增量同步：

1. `max(sync_time)` 读取卡顿：分区太多，PyIceberg 扫描所有分区的元数据时严重卡顿
2. **元数据更新扇出放大**：每次 17 条数据可能命中 5 个用户 × 3 个日期 = 15 个独立分区，CoW 模式下必须同时打开 15 个目录的旧小文件，各自剔除、各自写入新小文件，并修改 15 个 Manifest 记录
3. **乐观锁冲突锁死**：高频多线程并发修改庞大元数据树，极易触发乐观锁冲突，疯狂抛出 `CommitFailedException`
4. **物理重写毫无提升**：重写 1 个 50MB 未分区大文件 vs 重写 15 个几 KB 分区小文件，网络请求和事务提交的固有物理开销完全相同

## 解决方案

### 方案 1：完全不分区 + 追加写入

放弃细粒度分区，改用：
- 完全不分区表
- 追加写入（Append）而非覆盖（Overwrite）
- 技术性去重视图（[[risingwave]] 的 `ROW_NUMBER()`）

### 方案 2：流数据库缓存 + 湖格式归档

```
数据源  →  [[risingwave]]（实时缓存）  →  Iceberg Sink  →  [[oss]]（长周期归档）
           10 毫秒查询                    每 2 分钟大批次落湖
```

详见 [[risingwave]]。

## 结论

| 维度 | 问题根因 |
|------|----------|
| 读取慢 | 36.5 万分区 → metadata.json 膨胀至 GB 级 → PyIceberg 解析耗时 15s+ |
| 写入慢 | CoW 模式下少量数据命中大量分区 → Manifest 扇出修改 → 乐观锁冲突 → 指数重试 |
| 本质 | 小数据量（190 万行 / 50MB）不适合细粒度分区，分区带来的裁剪收益远不及元数据开销 |

## 参见

- [[iceberg]] — 表格式规范
- [[pyiceberg]] — Python 客户端
- [[risingwave]] — 流数据库解法
- [[iceberg-cow-problem]] — Copy-on-Write 问题
