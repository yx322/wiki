---
title: Lakekeeper
created: 2026-09-03
updated: 2026-09-03
type: entity
tags: [lakekeeper, iceberg, catalog, rest, docker]
sources: [raw/articles/cdc-risingwave-iceberg-lakekeeper-dialog.md]
confidence: high
contested: false
---

# Lakekeeper

自建的 [[iceberg-rest-catalog]] 实现（Rust 编写），管理 [[iceberg]] 表元数据，元数据存 PostgreSQL。是与阿里云 OSS Tables 托管服务相对的"标准协议"方案。

## 核心定位

- **只管元数据**：namespace、表、快照指针，不存数据文件（数据在 [[oss]]）
- **标准 Iceberg REST 协议**：DuckDB / PyIceberg / Spark 原生对接，无兼容性补丁
- **原子提交**：下游只看到完整快照，一致性由 Iceberg commit 保证

## 搭建要点（Docker）

| 步骤 | 说明 |
|------|------|
| 启动 | Docker 起 Lakekeeper + PG，REST API `http://lakekeeper:8181/catalog` |
| 建 Warehouse | Project + Warehouse，存储选 S3 类型（OSS endpoint / bucket / AK/SK） |
| ⚠️ 注册 server | **必须注册 server 并分配到 Warehouse**，否则写入失败（最常见坑） |
| 认证 | 本地开发可关；生产用 token / credential vending |

## 在 CDC 链路中的位置

```
MySQL binlog → RisingWave → Iceberg Sink → Lakekeeper（元数据）+ OSS（数据文件）
                                              ↑
                            DuckDB ATTACH ... (TYPE ICEBERG, REST 'http://lakekeeper:8181/catalog')
```

RisingWave Sink 配置：

```sql
catalog.type = 'rest',
catalog.uri = 'http://lakekeeper:8181/catalog',
warehouse = 'demo',
```

DuckDB 读端只有 30 行配置代码——对比 OSS Tables 方案需要子类化 Catalog 修协议（见 [[sync-on-query-vs-cdc]]）。

## 与 OSS Tables 托管服务对比

| 维度 | Lakekeeper（自建） | OSS Tables（托管） |
|------|------------------|-------------------|
| 协议 | 标准 Iceberg REST | 非标（无 /v1/config，SigV4 签名） |
| DuckDB 直连 | ✅ 原生 ATTACH | ❌ 只能 PyIceberg scan → Arrow 中转 |
| 元数据存储 | 自备 PostgreSQL | 托管 |
| 运维 | 自己部署（docker-compose 一个容器） | 零运维但协议坑自己填 |
| 小文件治理 | 自己负责（compaction） | 托管兜底 |

## 参见

- [[iceberg-rest-catalog]] — REST Catalog 协议职责
- [[task-data-pipeline]] — Lakekeeper 所在的完整 CDC 链路
- [[sync-on-query-vs-cdc]] — asw（OSS Tables）vs search-todo（Lakekeeper）架构对比
- [[iceberg]] — 表格式规范
- [[oss]] — 数据文件存储

^[raw/articles/cdc-risingwave-iceberg-lakekeeper-dialog.md]
