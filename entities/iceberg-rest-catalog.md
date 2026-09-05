---
title: Iceberg REST Catalog
type: entity
tags: [iceberg, catalog, rest-api]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/apache_iceberg_rest_catalog_pyiceberg.md
confidence: high
contested: false
---

# Iceberg REST Catalog

统一管理 [[iceberg]] 表指针的 HTTP 网关，不存储数据，只负责回答元数据请求。

## 核心定位

**比喻**：物流中心的前台系统 — 不存储货物，但负责登记货物位置、发放取货凭证、协调多用户访问。

在多用户、多计算引擎（Spark/Flink/Python）并发读写的复杂环境下，必须有一个"中央协调官"告诉大家：谁现在手里拿着最新的那份 `metadata.json` 账本指针？

## 职责

- **不负责存储真实数据文件**，只负责回答 HTTP 请求
- 接口是标准的（如 `GET /v1/namespaces/{ns}/tables/{table}`）
- 维护表的元数据指针（指向最新的 `metadata.json`）

## 标准 API 端点

```
GET  /v1/config                          # 获取配置
GET  /v1/namespaces                      # 列出命名空间
GET  /v1/namespaces/{ns}/tables          # 列出表
GET  /v1/namespaces/{ns}/tables/{table}  # 获取表详情
POST /v1/namespaces/{ns}/tables          # 创建表
POST /v1/namespaces/{ns}/tables/{table}/metrics  # 上报指标
```

## 写入协作流程

```
1. 计算引擎问 Catalog："我要写 user_table，可以吗？最新指针在哪？"
2. Catalog 验证权限后返回路径
3. 引擎在底层 OSS 写完 Parquet 文件
4. 引擎再次通过 HTTP 告诉 Catalog："我写完了，请把表指针安全地切到最新版本"
```

## 与 [[pyiceberg]] 的关系

PyIceberg 是 REST Catalog 的标准调用方：
- 初始化时默认去敲 `/v1/config` 的门，获取服务器配置
- 读写时按照标准 API 路径与 Catalog 通信

## 阿里云 OSS Tables 的兼容性问题

OSS Tables 没有完全实现标准 Iceberg REST Catalog 协议：

| 特性 | 标准 Iceberg REST | OSS Tables 实际 |
|------|------------------|----------------|
| Config 路径 | `GET /v1/config` | 无对应路径（返回 405） |
| List Tables | `GET /v1/namespaces/{ns}/tables` | `GET /tables/{arn}?namespace={ns}` |
| Get Table | `GET /v1/namespaces/{ns}/tables/{table}` | `GET /get-table?tableBucketARN=...` |
| 认证方式 | Bearer/OAuth2/Basic | AWS SigV4（服务名 `osstables`） |

详见 [[polars-iceberg-oss-tables]]。

## 参见

- [[iceberg]] — 表格式规范
- [[pyiceberg]] — Python 客户端
- [[polars-iceberg-oss-tables]] — OSS Tables 兼容性问题
- [[lakekeeper]] — 自建 REST Catalog 实现（标准协议方案）
