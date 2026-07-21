---
title: DuckDB
type: entity
tags: [duckdb, analytics, olap]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/DeltaLake_DuckDB_Polars_Guide.md
confidence: high
contested: false
---

# DuckDB

嵌入式 OLAP 数据库，无需服务器，`pip install` 即用，专为分析处理设计。

## 核心超能力

### 1. 向量化执行引擎

- 传统数据库（MySQL）一行一行看数据（吸管数米粒）
- DuckDB 一列一列看（一铲子几万条），分析统计快几十到几百倍

### 2. 万物皆可直接查

不需要导入，直接用 SQL 查文件：

```sql
SELECT * FROM 'sales.csv';
SELECT * FROM 'data.parquet';
SELECT * FROM read_parquet('s3://bucket/data/*.parquet');
```

跨界读取：甚至能直接连接云端 [[delta-lake]] / [[iceberg]] 数据。

### 3. 流式处理

8G 内存分析 20G 文件也不会爆内存，通过流式处理几秒算完。

## 典型应用场景

| 场景 | 痛点 | DuckDB 解法 |
|------|------|------------|
| 本地超大文件分析 | Excel 死机，Pandas 爆内存 | 向量化 + 流式处理 |
| 数据吸尘器 | 导入慢 | 直接查 CSV/Parquet/S3 |
| Python 办公自动化 | Pandas 代码复杂 | DataFrame 无缝转 SQL |
| 远程偷渡云端数据 | Spark 集群贵 | 谓词下推，只下载命中数据 |
| 嵌入式引擎 | 桌面软件缺 SQL | 缝进应用，立刻拥有 SQL 能力 |

## 谓词下推

云端 100TB 大表查"北京"，DuckDB 不会全下载，只捞属于"北京"的数据回本地算。省时、省钱、省带宽。

## 与 [[polars]] 的对比

| 维度 | DuckDB | Polars |
|------|--------|--------|
| 交互方式 | SQL 语句 | Python 链式调用 |
| 最擅长 | 跨文件查询、BI 报表 | 数据清洗、特征工程 |
| 比喻 | 自动售货机 | 开放式厨房 |
| 底层 | Apache Arrow | Apache Arrow |

两者底层都遵循 Apache Arrow 标准，数据传输零成本。

## 在湖仓架构中的角色

```
[[delta-lake]] / [[iceberg]]  →  [[duckdb]]  →  [[polars]]
   存数据在云端                    快速吸出数据         清洗和 ML 准备
```

## 参见

- [[polars]] — 数据处理搭档
- [[lakehouse]] — 湖仓一体架构
- [[iceberg]] — 可直接查询
