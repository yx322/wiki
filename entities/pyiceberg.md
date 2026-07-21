---
title: PyIceberg
type: entity
tags: [pyiceberg, iceberg, python]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/apache_iceberg_rest_catalog_pyiceberg.md
  - raw/articles/polars_write_iceberg_oss_tables.md
confidence: high
contested: false
---

# PyIceberg

专为 Python 生态打造的纯 Python 客户端，操作 [[iceberg]] 表的轻量级执行工具。

## 核心定位

过去操作 Iceberg 必须启动庞大的 JVM（Spark/Flink）。PyIceberg 是纯 Python 客户端，无需 JVM。

**比喻**：物流中心里的无人平衡车 — 按照包装规范装箱，通过前台系统登记，然后去货架搬运货物。

## 职责

当执行 `table.append(df.to_arrow())` 时，PyIceberg 在 Python 内存里：

1. 把数据切片
2. 计算 Max/Min 统计信息
3. 封装成符合 Iceberg 规范的 Parquet 文件和清单文件
4. 上传到 OSS 存储桶
5. 通过 REST Catalog 提交元数据变更

## 支持的认证方式

- Bearer token（`token` 参数）
- OAuth2（`credential` 参数）
- Basic auth（`header.XXX` 参数）

**不支持**：AWS SigV4、阿里云 OSS4-HMAC-SHA256

## 与 REST Catalog 的交互

```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog(
    "oss_tables",
    type="rest",
    uri="https://region.oss-tables.aliyuncs.com/iceberg",
    warehouse="acs:osstables:region:uid:bucket/name",
    **{
        "s3.endpoint": "https://oss-region.aliyuncs.com",
        "s3.access-key-id": ak,
        "s3.secret-access-key": sk,
    },
)

table = catalog.load_table(("namespace", "table_name"))
table.append(df.to_arrow())
```

## 性能瓶颈

PyIceberg 在以下场景会严重退化：

### 元数据解析卡死

190 万行数据被切碎为 36.5 万个分区时，`metadata.json` 膨胀至 GB 级。PyIceberg 单线程解析这个庞大元数据，光解析就卡住 15 秒以上，还没开始下载数据。

详见 [[iceberg-small-file-problem]]。

### 写入锁死

Copy-on-Write 模式下，高频小批量写入（17~150 条）会触发：
- 元数据扇出修改（15 个分区同时更新）
- 乐观锁冲突（`CommitFailedException`）
- 指数级退避重试

详见 [[iceberg-cow-problem]]。

## 与 OSS Tables 的兼容性

PyIceberg 无法直接使用阿里云 OSS Tables，因为：

1. OSS Tables 不支持标准 Iceberg REST 协议（`GET /v1/config` 返回 405）
2. 认证方式不兼容（OSS Tables 需要 AWS SigV4，PyIceberg 不支持）
3. warehouse 数据存储在用户无法访问的内部 bucket

详见 [[polars-iceberg-oss-tables]]。

## 参见

- [[iceberg]] — 表格式规范
- [[iceberg-rest-catalog]] — REST 网关
- [[iceberg-small-file-problem]] — 性能陷阱
- [[polars-iceberg-oss-tables]] — OSS Tables 兼容性问题
