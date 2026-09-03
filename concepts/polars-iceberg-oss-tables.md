---
title: Polars write_iceberg 与 OSS Tables 兼容性问题
type: concept
tags: [polars, iceberg, oss, compatibility, pyiceberg]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/polars_write_iceberg_oss_tables.md
confidence: high
contested: false
---

# Polars write_iceberg 与 OSS Tables 兼容性问题

Polars 原生 `write_iceberg()` 无法直接写入阿里云 OSS Tables，因为协议不兼容。

## 问题现象

尝试使用 Polars `write_iceberg()` 写入 OSS Tables 时，所有方案都失败：

| 方案 | 失败原因 |
|------|---------|
| PyIceberg REST catalog + table.append() | 405 Method Not Allowed |
| Polars write_iceberg() + catalog_options | TypeError: 参数不存在 |
| Polars write_iceberg() + SqlCatalog | ACCESS_DENIED: 内部 bucket 无法访问 |
| REST catalog 读取 + SqlCatalog 写入 | REST catalog 加载表时 405 |

## 根本原因

**OSS Tables 没有实现标准 Iceberg REST Catalog 协议，而是使用自定义 API。**

### API 对比

| 特性 | 标准 Iceberg REST | OSS Tables 实际 |
|------|------------------|----------------|
| Endpoint 格式 | `{region}.oss-tables.aliyuncs.com` | `{bucket}-{uid}.{region}.oss-tables.aliyuncs.com` |
| 认证方式 | AWS SigV4（服务名 `osstables`） | OSS4-HMAC-SHA256（阿里云签名） |
| Config 路径 | `GET /v1/config?warehouse=s3://...` | 无对应路径（405） |
| List Tables | `GET /v1/namespaces/{ns}/tables` | `GET /tables/{arn}?namespace={ns}` |
| Get Table | `GET /v1/namespaces/{ns}/tables/{table}` | `GET /get-table?tableBucketARN=...` |
| 参数格式 | `warehouse=s3://bucket/ns` | `tableBucketARN=acs:osstables:region:uid:bucket/name` |

### PyIceberg 的认证限制

PyIceberg 的 `RestCatalog` 只支持：
- Bearer token（`token` 参数）
- OAuth2（`credential` 参数）
- Basic auth（`header.XXX` 参数）

**不支持 AWS SigV4**！即使 OSS Tables 支持 SigV4，PyIceberg 也不会发送签名，导致 403 错误。

### FileIO vs Catalog 问题

两层都有问题：

- **Catalog 层面**：`GET /v1/config` 返回 405，说明 OSS Tables 根本没有实现标准 Iceberg REST Catalog 的 `/v1/config` 端点。PyIceberg 在初始化时就会失败，根本走不到 FileIO 层。
- **FileIO 层面**：即使 Catalog 能工作，warehouse 数据存储在内部 bucket（`974e342d-e7ea-4088-4ppsf723d551qiwbnydj50r0an50cqf7o--table-oss`），用户的 AK/SK 无法直接访问，FileIO 也会失败。

## 解决方案

### 方案 A：Spark 写入（官方推荐）

```python
# pyspark + aliyun oss-tables connector
df.write.format("iceberg").save("task.task_records")
```

### 方案 B：手动构造 Iceberg 格式

1. 写 Parquet 到 `s3://xmh-ai-data/task/data/`（用户自己的 bucket）
2. 生成标准 Avro manifest 文件
3. 生成完整的 Iceberg V2 metadata JSON
4. 上传到 OSS
5. 用 SDK 的 `update_table_metadata_location` 提交元数据变更

**注意**：需要生成标准 Avro 格式的 manifest（不是 JSON），否则 OSS Tables 可能无法识别。

### 方案 C：联系阿里云

确认 OSS Tables 的正确写入方式和 API 文档，或者请求提供标准 AWS SigV4/OAuth2 认证选项。

### 方案 D：使用 requests-aws4auth

```python
from requests_aws4auth import AWS4Auth
import boto3

session = boto3.Session(
    aws_access_key_id=ak,
    aws_secret_access_key=sk,
    region_name=OSS_REGION,
)
credentials = session.get_credentials()
auth = AWS4Auth(
    credentials.access_key,
    credentials.secret_key,
    OSS_REGION,
    'osstables',  # 服务名
    session_token=credentials.token,
)

catalog = load_catalog(
    "oss_tables",
    type="rest",
    uri="https://cn-hangzhou.oss-tables.aliyuncs.com/iceberg",
    warehouse="acs:osstables:cn-hangzhou:1842997423053480:bucket/xmh-ai-data",
    **{
        "header.Authorization": auth,
        "s3.endpoint": f"https://oss-{OSS_REGION}.aliyuncs.com",
        "s3.access-key-id": ak,
        "s3.secret-access-key": sk,
    },
)
```

**注意**：即使认证通过，OSS Tables 只实现了部分 Iceberg REST API（如 `/v1/config`），其他端点（如 `/v1/namespaces`）可能返回 405。需要进一步测试完整的读写流程。

## 结论

Polars 原生 `write_iceberg()` 无法直接写入 OSS Tables，因为：
1. OSS Tables 不支持标准 Iceberg REST 协议
2. SDK 没有数据写入 API
3. warehouse 数据存储在用户无法访问的内部 bucket

建议使用 Spark 或手动构造 Iceberg 格式（需要生成标准 Avro manifest）。

## 参见

- [[iceberg]] — 表格式规范
- [[pyiceberg]] — Python 客户端
- [[iceberg-rest-catalog]] — REST 网关
- [[oss]] — 阿里云对象存储
