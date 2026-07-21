# Polars write_iceberg 写入 OSS Tables 问题记录

## 背景

尝试使用 Polars 原生 `write_iceberg()` 方法将数据写入阿里云 OSS Tables（Iceberg 格式）。

## 方案演进

### 方案 1：PyIceberg REST catalog + table.append()

**代码**：
```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog(
    "oss_tables",
    type="rest",
    uri="https://cn-hangzhou.oss-tables.aliyuncs.com",
    warehouse="s3://xmh-ai-data/task",
    **{
        "s3.endpoint": "https://oss-cn-hangzhou.aliyuncs.com",
        "s3.access-key-id": ak,
        "s3.secret-access-key": sk,
    },
)

table = catalog.load_table(("task", "task_records"))
table.append(df.to_arrow())  # 405 错误
```

**问题**：`RESTError 405: Method Not Allowed`

**原因**：OSS Tables 不支持 PyIceberg 的 REST catalog 协议（`GET /v1/config` 返回 405）

---

### 方案 2：Polars write_iceberg() + catalog_options

**代码**：
```python
df.write_iceberg(
    "task.task_records",
    catalog_options={
        "type": "rest",
        "uri": "https://cn-hangzhou.oss-tables.aliyuncs.com",
        "warehouse": "s3://xmh-ai-data/task",
        ...
    },
    mode="append",
)
```

**问题**：`TypeError: DataFrame.write_iceberg() got an unexpected keyword argument 'catalog_options'`

**原因**：Polars 1.41.2 的 `write_iceberg()` 签名是 `(target, mode)`，不支持 `catalog_options` 参数

---

### 方案 3：Polars write_iceberg() + SqlCatalog Table 对象

**代码**：
```python
from pyiceberg.catalog.sql import SqlCatalog

catalog = SqlCatalog(
    "local",
    uri="sqlite:///local_catalog.db",
    warehouse="s3://974e342d-e7ea-4088-4ppsf723d551qiwbnydj50r0an50cqf7o--table-oss",
    **{
        "s3.endpoint": "https://oss-cn-hangzhou.aliyuncs.com",
        "s3.access-key-id": ak,
        "s3.secret-access-key": sk,
    },
)

table = catalog.load_table(("task", "task_records"))
df.write_iceberg(table, mode="append")
```

**问题**：`ACCESS_DENIED: No response body`

**原因**：warehouse 路径指向 OSS Tables 的内部 bucket（`974e342d-e7ea-4088-4ppsf723d551qiwbnydj50r0an50cqf7o--table-oss`），用户的 AK/SK 无法直接访问

---

### 方案 4：REST catalog 读取 + SqlCatalog 写入

**思路**：
1. 用 REST catalog 读取 metadata（通过 API）
2. 用 SqlCatalog 注册本地表
3. 用 Polars write_iceberg 写入

**问题**：REST catalog 加载表时仍然返回 405

**原因**：OSS Tables 的 REST API 不是标准 Iceberg REST 协议，所有 `/v1/*` 路径都返回 405

---

## 根本原因：OSS Tables 使用自定义 API，非标准 Iceberg REST Catalog

**核心问题：OSS Tables 没有实现标准 Iceberg REST Catalog 协议，而是使用自定义 API。**

### API 对比

| 特性 | 标准 Iceberg REST Catalog | OSS Tables 实际 API |
|------|--------------------------|---------------------|
| **Endpoint 格式** | `{region}.oss-tables.aliyuncs.com` | `{bucket}-{uid}.{region}.oss-tables.aliyuncs.com` |
| **认证方式** | AWS SigV4（服务名 `osstables`） | OSS4-HMAC-SHA256（阿里云签名） |
| **Config 路径** | `GET /v1/config?warehouse=s3://...` | 无对应路径 |
| **List Tables 路径** | `GET /v1/namespaces/{ns}/tables` | `GET /tables/{arn}?namespace={ns}` |
| **Get Table 路径** | `GET /v1/namespaces/{ns}/tables/{table}` | `GET /get-table?tableBucketARN=...&namespace=...&name=...` |
| **参数格式** | `warehouse=s3://bucket/ns` | `tableBucketARN=acs:osstables:region:uid:bucket/name` |

### SDK 实际调用示例

通过拦截 SDK 的 HTTP 请求，发现实际调用格式：

```
GET https://xmh-ai-data-1842997423053480.cn-hangzhou.oss-tables.aliyuncs.com/tables/acs%3Aosstables%3Acn-hangzhou%3A1842997423053480%3Abucket%2Fxmh-ai-data?namespace=task

Headers:
  Authorization: OSS4-HMAC-SHA256 Credential=LTAI.../20260625/cn-hangzhou/osstables/aliyun_v4_request,Signature=...
  x-oss-date: 20260625T064401Z
  x-oss-content-sha256: UNSIGNED-PAYLOAD
```

### 最新发现：OSS Tables REST Catalog 部分支持 Iceberg REST API + AWS SigV4

通过测试发现：

1. **正确的 endpoint**：`https://{region}.oss-tables.aliyuncs.com/iceberg`（带 `/iceberg` 前缀）
2. **正确的 warehouse 格式**：只有 bucket ARN，不含 namespace
   ```
   warehouse=acs:osstables:cn-hangzhou:1842997423053480:bucket/xmh-ai-data
   ```
3. **认证方式**：AWS SigV4（服务名 `osstables`）

**验证过程**：
- `/v1/config` + AWS SigV4 → 403 "SignatureDoesNotMatch"（签名实现有 bug，但说明 SigV4 被支持）
- `/v1/namespaces` → 405（端点不存在，部分实现 Iceberg REST）
- 无认证头 → 403 "Anonymous user has no right to access this bucket."

**关键发现**：OSS Tables REST Catalog **部分支持** Iceberg REST API，并使用 AWS SigV4 认证（服务名 `osstables`）。

### 为什么 PyIceberg 仍然失败

PyIceberg 的 `RestCatalog` **默认不发送任何认证头**，它只支持：
- Bearer token（`token` 参数）
- OAuth2（`credential` 参数）
- Basic auth（`header.XXX` 参数）

**不支持 AWS SigV4**！

所以即使 OSS Tables 支持 SigV4，PyIceberg 也不会发送签名，导致 403 错误。

### 解决方案：使用 requests-aws4auth

安装依赖：
```bash
pip install requests-aws4auth boto3
```

配置 PyIceberg 使用 SigV4 认证：
```python
from pyiceberg.catalog import load_catalog
from requests_aws4auth import AWS4Auth
import boto3

# 创建 SigV4 认证
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

# 配置 PyIceberg 使用 SigV4 认证
catalog = load_catalog(
    "oss_tables",
    type="rest",
    uri="https://cn-hangzhou.oss-tables.aliyuncs.com/iceberg",
    warehouse="acs:osstables:cn-hangzhou:1842997423053480:bucket/xmh-ai-data",
    **{
        "header.Authorization": auth,  # 关键：添加 SigV4 认证
        "s3.endpoint": f"https://oss-{OSS_REGION}.aliyuncs.com",
        "s3.access-key-id": ak,
        "s3.secret-access-key": sk,
    },
)
```

**注意**：即使认证通过，OSS Tables 只实现了部分 Iceberg REST API（如 `/v1/config`），其他端点（如 `/v1/namespaces`）可能返回 405。需要进一步测试完整的读写流程。

### 可行的替代方案

1. **Spark + OSS Tables connector**（官方推荐）
   ```python
   # pyspark + aliyun oss-tables connector
   df.write.format("iceberg").save("task.task_records")
   ```

2. **手动构造 Iceberg 格式**（复杂但可行）
   - 写 Parquet 到 `s3://xmh-ai-data/task/data/`（用户自己的 bucket）
   - 生成标准 Avro manifest 文件
   - 生成完整的 Iceberg V2 metadata JSON
   - 上传到 OSS
   - 用 SDK 的 `update_table_metadata_location` 提交元数据变更

3. **联系阿里云**
   确认 OSS Tables 的正确写入方式和 API 文档，或者请求提供标准 AWS SigV4/OAuth2 认证选项。

### 验证过程

测试了多种路径组合：

```
# 标准 Iceberg REST 路径（全部 405）
GET https://{region}.oss-tables.aliyuncs.com/v1/config                        → 405
GET https://{region}.oss-tables.aliyuncs.com/v1/namespaces                    → 405
GET https://{region}.oss-tables.aliyuncs.com/v1/namespaces/task/tables        → 405

# 带 bucket uid 的 endpoint（SDK 格式）
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/v1/config         → 405
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/v1/namespaces     → 405

# 带 /iceberg 前缀（部分存在但不兼容）
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/iceberg/v1/config
  → 400 "The specified table bucket ARN is not valid."
  （注意：返回 400 而非 405，说明这个路径存在，但参数格式不对）
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/iceberg/v1/namespaces            → 405
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/iceberg/v1/namespaces/task/tables → 405

# SDK 实际使用的路径（正常工作）
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/tables/{arn}?namespace={ns}      → 200 ✓
GET https://{bucket}-{uid}.{region}.oss-tables.aliyuncs.com/get-table?tableBucketARN=...     → 200 ✓
```

### FileIO vs Catalog 问题

有观点认为问题出在 FileIO 而非 Catalog：

> "REST Catalog 正常，但 bucket 鉴权失败，所以 PyIceberg 不兼容"

但实际上：
- **Catalog 层面**：`GET /v1/config` 返回 405，说明 OSS Tables 根本没有实现标准 Iceberg REST Catalog 的 `/v1/config` 端点。PyIceberg 在初始化时就会失败，根本走不到 FileIO 层。
- **FileIO 层面**：即使 Catalog 能工作，warehouse 数据存储在内部 bucket（`974e342d-e7ea-4088-4ppsf723d551qiwbnydj50r0an50cqf7o--table-oss`），用户的 AK/SK 无法直接访问，FileIO 也会失败。

**两层都有问题**，但 Catalog 是第一道关卡，直接阻止了所有后续操作。

### `/iceberg/v1/config` 的线索

`GET /iceberg/v1/config` 返回 **400**（而非 405），说明 OSS Tables 可能在 `/iceberg/` 前缀下实现了一部分 Iceberg REST API。但：
- 参数格式不同（需要 `tableBucketARN` 而非 `warehouse`）
- 其他路径（`/iceberg/v1/namespaces`）仍然返回 405
- PyIceberg 不会使用 `/iceberg/` 前缀，它只请求 `/v1/config`

这意味着即使 OSS Tables 有部分 Iceberg REST 支持，PyIceberg 也无法直接使用，因为路径不匹配。

### 调用链路分析

所有失败的方案，最终都指向同一个问题：

```
用户代码（Polars write_iceberg 或 PyIceberg table.append）
  ↓
PyIceberg RestCatalog.__init__()
  ↓
self._fetch_config()
  ↓
GET https://{region}.oss-tables.aliyuncs.com/v1/config?warehouse=s3://...
  ↓
HTTP 405 Method Not Allowed（OSS Tables 没有这个路径）
  ↓
✗ 全部失败
```

### 各方案失败原因对照

| 方案 | 用户代码 | 实际调用 | 失败原因 |
|------|---------|---------|---------|
| 方案 1 | `table.append(df)` | PyIceberg REST catalog → `GET /v1/config` | 405（路径不存在） |
| 方案 2 | `df.write_iceberg(...)` | Polars 内部创建 PyIceberg REST catalog | 参数不存在 + 405 |
| 方案 3 | `df.write_iceberg(table)` | SqlCatalog → 访问内部 bucket | ACCESS_DENIED |
| 方案 4 | REST 读取 + SqlCatalog 写入 | PyIceberg REST catalog → `GET /v1/config` | 405 |

**结论：不管用 Polars 还是直接用 PyIceberg，只要走 REST catalog，都会因为 OSS Tables 使用自定义 API 而非标准 Iceberg REST 协议而失败。**

## 可行的替代方案

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

**注意**：需要生成标准 Avro 格式的 manifest（不是 JSON），否则 OSS Tables 可能无法识别

### 方案 C：联系阿里云

确认 OSS Tables 的正确写入方式和 API 文档

## 依赖安装

```bash
pip install polars pyarrow
pip install "pyiceberg[sql]" sqlalchemy  # SqlCatalog 需要
```

## 测试脚本

- `test_polars_iceberg.py` — Polars write_iceberg 测试
- `test_sql_catalog.py` — SqlCatalog 测试
- `test_rest_api.py` — REST API 路径测试
- `test_sdk_api.py` — SDK API 测试
- `test_manual_write.py` — 手动构造 Iceberg 格式测试

## 结论

Polars 原生 `write_iceberg()` 无法直接写入 OSS Tables，因为：
1. OSS Tables 不支持标准 Iceberg REST 协议
2. SDK 没有数据写入 API
3. warehouse 数据存储在用户无法访问的内部 bucket

建议使用 Spark 或手动构造 Iceberg 格式（需要生成标准 Avro manifest）。
