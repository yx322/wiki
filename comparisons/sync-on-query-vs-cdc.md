---
title: Sync-on-Query vs CDC（asw vs search-todo）
created: 2026-09-03
updated: 2026-09-03
type: comparison
tags: [cdc, sync-on-query, pyiceberg, risingwave, lakekeeper, oss-tables, architecture]
sources: [raw/articles/cdc-risingwave-iceberg-lakekeeper-dialog.md]
confidence: high
contested: false
---

# Sync-on-Query vs CDC

两种 Iceberg 数据入库架构的对比：**asw（search-workorder，Sync-on-Query 客户端写入）** vs **search-todo（CDC 流式链路写入）**。

核心差别一句话：asw 是"查的时候自己拉数据自己写"，search-todo 是"流式链路替你写好，你只管读"。

## 全维度对比

| 维度 | asw (search-workorder) | search-todo |
|---|---|---|
| Catalog | 阿里云 OSS Tables 托管服务（ARN + SigV4） | 自建 [[lakekeeper]]（PG 存元数据） |
| 谁写入 | skill 自己的 Python 代码（[[pyiceberg]]） | [[risingwave]] CDC，skill 不写 |
| 同步时机 | Sync-on-Query：查询时调 ASW API，按上次 sync_time 增量拉，按 id Upsert | MySQL binlog 持续同步，秒级 |
| 数据新鲜度 | 取决于上次查询（两次查询之间是旧的） | 准实时 |
| 写方式 | PyIceberg append/overwrite（不支持 MoR 删除文件，更新 = 整表覆盖） | RisingWave upsert sink，merge-on-read，只写增量文件 |
| 读路径 | DuckDB iceberg 扩展连不上 OSS Tables（SigV4 signing-name 不兼容），只能 PyIceberg scan → Arrow → 注册进 DuckDB | 标准 REST catalog，DuckDB 直接 `ATTACH ... TYPE iceberg` |

## asw 的补偿性代码（全非业务逻辑）

```
OssTablesRestCatalog 子类化        → 修托管服务的协议非标（绕过坏掉的 /v1/config 端点）
双路凭证（client.* / s3.*）        → REST 层和数据层认证割裂
AWS_REQUEST_CHECKSUM_CALCULATION=when_required → 绕 OSS 不支持的 checksum trailer
Arrow schema 严格对齐              → required 字段非 nullable，否则静默失败
Sync-on-Query + 按 id Upsert       → 用 Python 模拟 CDC 语义
```

相关协议细节见 [[iceberg-rest-catalog]] 的 OSS Tables 兼容性一节、[[polars-iceberg-oss-tables]]。

## search-todo 的读端

Lakekeeper 是标准 Iceberg REST 协议，DuckDB 原生支持，读端只有 30 行配置代码，上述坑全没有。链路细节见 [[task-data-pipeline]]。

## 取舍总结

> asw：用 Python 在客户端手搓了一个不完整的流式同步引擎，来迁就托管服务的协议缺陷。
> search-todo：把同步、写入、目录三件事各交给专职组件，应用层只剩"读"。
> 前者的复杂度藏在代码里，后者的复杂度摆在 docker-compose 里（RisingWave + Lakekeeper + PG 三个容器）——后者的可维护性高一个量级。

**本质**：search-todo 不是没有运维成本，而是把成本从"代码复杂度"换成了"组件运维"。前者在每次协议升级/边界 case 时都要人肉补丁，后者是声明式配置一次到位。对数据量会增长、多用户查询的系统，这笔交换是划算的。

## 决策要点

- 源头是自有 MySQL → **CDC 优先**（binlog 是数据库亲口给的变更信号，见 [[cdc-vs-api-sync]]）
- 源头是第三方 API（无 binlog，如 OA 接口）→ 只能拉接口；此时 Sync-on-Query 还是定时拉，取决于新鲜度要求
- 数据量小、单机查询 → 复杂度可接受；数据量会涨、多用户查询 → 组件化链路回本

## 参见

- [[task-data-pipeline]] — search-todo 完整链路
- [[lakekeeper]] — 自建 REST Catalog
- [[iceberg-rest-catalog]] — REST 协议与 OSS Tables 非标问题
- [[risingwave]] — 流式引擎
- [[cdc-vs-api-sync]] — CDC vs API 拉取的模式级对比
- [[iceberg-cow-problem]] — upsert 写放大问题

^[raw/articles/cdc-risingwave-iceberg-lakekeeper-dialog.md]
