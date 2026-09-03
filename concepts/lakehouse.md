---
title: 湖仓一体 (Lakehouse)
type: concept
tags: [lakehouse, oss, iceberg, delta-lake, architecture]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/DeltaLake_DuckDB_Polars_Guide.md
  - raw/articles/湖仓一体化（纯接口模式）性能重构落地方案.md
  - raw/articles/湖仓一体化性能重构落地方案.md
  - raw/articles/阿里云OSS与CAP定理.md
confidence: high
contested: false
---

# 湖仓一体 (Lakehouse)

将表格式叠加在廉价对象存储上，查询引擎按需启动。目标是用廉价对象存储替代专用集群存储，计算节点按需扩展。

## 架构

```
[[oss]]（对象存储）  →  [[iceberg]] / [[delta-lake]]（表格式）  →  [[duckdb]] / [[polars]] / [[risingwave]]（计算）
  廉价、无限扩展           元数据管理、ACID 事务                    按需启动、弹性计算
```

## 核心优势

| 维度 | 传统数仓 | 湖仓一体 |
|------|---------|---------|
| 存储成本 | 专用集群存储，昂贵 | 对象存储，低一到两个数量级 |
| 扩展方式 | 加节点，成本高 | 计算和存储独立扩展 |
| 运维 | 需要运维存储集群 | 无需运维，云厂商托管 |
| 容灾 | 自建多副本 | 原生多副本，跨可用区 |

## 与 MPP 分析型数据库的对比

| 维度 | MPP（ClickHouse/Snowflake） | Lakehouse |
|------|---------------------------|-----------|
| 存储 | 专用集群存储 | 对象存储（S3/OSS） |
| 扩展 | 加节点 | 计算和存储独立扩展 |
| 延迟 | 亚秒级 | 秒级到分钟级 |
| 成本 | 高（集群常驻） | 低（按需启动） |

## 为什么 Lakehouse 更有前途

对象存储相对文件存储的优势是碾压性的：
- 成本低一到两个数量级
- 无限弹性扩展
- 原生多副本容灾
- 无需运维存储集群

对象存储的短板是延迟和元数据：单次请求延迟高于本地磁盘，目录列举在海量文件下性能退化。但 OLAP 恰恰对延迟不敏感 — 用户提交查询后等待数秒到数分钟是可接受的。

Lakehouse 只需解决元数据问题（通过表格式的 metadata layer 或 REST Catalog 提供高效快照和文件索引），即可继承对象存储对文件存储的全部碾压性优势。

## 典型架构：RisingWave + Iceberg

```
数据源（MySQL CDC / API）  →  [[risingwave]]（流数据库）  →  [[iceberg]] Sink  →  [[oss]]
                              实时计算、物化视图               每 2 分钟大批次落湖
```

详见 [[risingwave]]。

## 参见

- [[iceberg]] — 表格式规范
- [[delta-lake]] — 竞争表格式
- [[oss]] — 存储底座
- [[risingwave]] — 流数据库
