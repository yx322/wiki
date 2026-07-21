---
title: Delta Lake
type: entity
tags: [delta-lake, lakehouse, oss, storage]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/DeltaLake_DuckDB_Polars_Guide.md
confidence: high
contested: false
---

# Delta Lake

存储架构（存储层框架），在对象存储之上引入事务日志（`_delta_log`）管理 Parquet 数据文件。

## 核心定位

Delta Lake **本身不存储数据**，是在 S3/OSS/HDFS 之上加装的一套软件管理框架。通过引入事务日志目录，让原本杂乱无章的数据湖文件拥有 ACID 事务和版本控制能力。

## 简单理解

```
OSS（阿里云） = 硬盘（存文件的）
Delta Lake   = 文件组织方式（让普通文件变成一张表）
DuckDB       = 读文件的程序
```

Delta Lake 是一种**存储在对象存储上的表格式**——它不是数据库，不是存储引擎，而是定义"一堆 Parquet 文件怎么组织成一张表"的规范。

## 物理存储结构

OSS bucket 里实际存的是一堆 **Parquet 文件** + 一个 **_delta_log** 目录：

```
s3://xmh-analysis/task_data/task_records/
├── _delta_log/
│   ├── 00000000000000000000.json   ← 第1次写入的元数据
│   ├── 00000000000000000001.json   ← 第2次写入的元数据
│   └── ...
├── part-00000-xxx.snappy.parquet   ← 实际数据文件
├── part-00001-xxx.snappy.parquet
└── ...
```

- **Parquet 文件** = 实际数据（列式存储，压缩率高，查询快）
- **_delta_log** = 事务日志（记录了每次写了哪些文件、删了哪些文件）

## 与直接存 Parquet 的区别

| | 直接存 Parquet | Delta Lake |
|---|---|---|
| 存储 | 一堆 parquet 文件 | 同上 + _delta_log |
| 追加写入 | ❌ 会覆盖或冲突 | ✅ 事务保证，ACID |
| 更新/删除 | ❌ 要重写整个文件 | ✅ 按行 upsert |
| 时间旅行 | ❌ | ✅ 可以查历史版本 |
| Schema 演变 | ❌ | ✅ 支持加字段 |

## 数据查询链路

```
大模型写 SQL
    ↓
DuckDB 读 _delta_log → 知道有哪些 parquet 文件
    ↓ 通过 S3 协议
OSS 上拉 parquet 文件
    ↓
执行 SQL → 返回结果
```

整个过程中数据存在阿里云 [[oss]] 上，Delta Lake 只是定义这些文件怎么组织、怎么保证一致性。[[duckdb]] 通过 S3 协议直接读取，不需要额外的服务进程。

## 三大法宝

| 法宝 | 能力 | 比喻 |
|------|------|------|
| 总账本 | ACID 事务 | 所有读写必须登记，崩溃时未完成操作被忽略 |
| 后悔药 | 版本控制 / 时间旅行 | 记录每一分钟变化，可穿越到任意时间点 |
| 安检口 | Schema 强制 | 门口安检，数据类型不符直接退货 |

## 与 [[iceberg]] 的对比

| 维度 | Delta Lake | Iceberg |
|------|-----------|---------|
| 元数据格式 | JSON 事务日志 | metadata.json + manifest 树 |
| 生态 | Spark/Databricks 主导 | 多引擎中立（Spark/Flink/Trino/PyIceberg） |
| 云服务 | Databricks Delta | 阿里云 OSS Tables、AWS Glue |
| 分区演化 | 不支持 | 支持 |

## 生态三位一体

```
存储体（S3/OSS）  →  Delta Lake（管理系统）  →  计算引擎（Spark/Flink）
  提供廉价空间          画线、做标记、做账本         听从指挥进行计算
```

## 参见

- [[iceberg]] — 竞争表格式
- [[lakehouse]] — 湖仓一体架构
- [[oss]] — 对象存储（Delta Lake 的物理底座）
- [[duckdb]] — 可直接查询 Delta Lake
- [[delta-lake]] — 本页自身
