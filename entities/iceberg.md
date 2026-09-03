---
title: Apache Iceberg
type: entity
tags: [iceberg, lakehouse, oss, storage]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/apache_iceberg_rest_catalog_pyiceberg.md
  - raw/articles/iceberg-storage-architecture-and-optimization.md
  - raw/articles/Iceberg表性能退化与架构优化分析.md
confidence: high
contested: false
---

# Apache Iceberg

开放表格式规范，在对象存储（S3/OSS）之上提供 ACID 事务、schema 演化、分区演化和时间旅行。

## 核心定位

Iceberg 本身**不是软件**，是一套**表格式规范**。它规定了 Parquet/ORC 数据文件在分布式对象存储上如何组织，才能被当成可执行 SELECT/INSERT/UPDATE 的数据库表。

## 元数据树结构

```
metadata.json          ← 表元数据（记录快照与 manifest-list 路由）
  ↓
manifest-list          ← 存储 N 个 manifest 文件的元数据（含分区范围统计）
  ↓
manifest-1 ~ manifest-N ← 存储数据文件的绝对路径、分区键值及列级 Min/Max 统计
  ↓
data-1.parquet ~ data-N.parquet ← 散落在对象存储中的实际数据文件
```

## 核心能力

| 能力 | 说明 |
|------|------|
| ACID 事务 | 多引擎安全读写，乐观锁并发控制 |
| 时间旅行 | 查询历史版本数据 |
| Schema 演化 | 加减列、改类型，不影响已有数据 |
| 分区演化 | 改分区策略，旧数据无需重写 |
| 行级定位 | 通过 Max/Min 统计快速过滤 |

## 三组件协作

Iceberg 生态由三层组件构成：

1. **Iceberg 规范** — 定义元数据树结构（静态规范）
2. **[[iceberg-rest-catalog]]** — 统一管理表指针的 HTTP 网关（控制面）
3. **[[pyiceberg]]** — Python 轻量级客户端（执行层）

## 读取流程

```
用户代码
  ↓
PyIceberg 调用 REST Catalog（GET /v1/namespaces/{ns}/tables/{table}）
  ↓
Catalog 返回 metadata.json 指针
  ↓
PyIceberg 读取 metadata.json → manifest-list → manifest-file
  ↓
PyIceberg 根据分区裁剪 + Min/Max 过滤，只下载命中文件
  ↓
返回 Arrow Table 给用户
```

## 写入流程

```
用户代码（table.append(df)）
  ↓
PyIceberg 计算数据统计信息（Max/Min/分区键）
  ↓
生成 Parquet 数据文件 → 上传到 OSS
  ↓
生成 manifest 文件 → 上传到 OSS
  ↓
生成新 metadata.json → 上传到 OSS
  ↓
调用 REST Catalog 提交元数据变更（原子切换指针）
  ↓
写入完成
```

## 性能陷阱

Iceberg 在小数据量 + 过度分区场景下会严重退化，详见 [[iceberg-small-file-problem]]。

## 与 OSS Tables 的兼容性

阿里云 OSS Tables 使用自定义 API，不完全兼容标准 Iceberg REST Catalog 协议。详见 [[polars-iceberg-oss-tables]]。

## 参见

- [[delta-lake]] — 竞争表格式
- [[lakehouse]] — 湖仓一体架构
- [[pyiceberg]] — Python 客户端
- [[iceberg-rest-catalog]] — REST 网关
