# MySQL CDC → RisingWave → Iceberg（Lakekeeper + OSS）技术对话记录

> 来源：chat.z.ai 对话记录（2026-09-03 整理）
> 主题：CDC 实时入湖链路的完整搭建 + 组件职责 + asw/search-todo 双架构对比

---

## 目录

1. [整体架构](#1-整体架构)
2. [各组件职责](#2-各组件职责)
3. [搭建步骤](#3-搭建步骤)
4. [Polars / DuckDB 在链路中的角色](#4-polars--duckdb-在链路中的角色)
5. [关键设计决策](#5-关键设计决策)
6. [常见坑清单](#6-常见坑清单)
7. [架构对比：asw vs search-todo](#7-架构对比asw-vs-search-todo)

---

## 1. 整体架构

```
MySQL (binlog, ROW 格式)
│ CDC (Debezium 内嵌于 RisingWave)
▼
RisingWave
├── Source: mysql-cdc（快照 + 增量）
├── 处理: 物化视图 / SQL（过滤、join、聚合、去重）—— ★ 存入前的处理在这里做
└── Iceberg Sink（upsert 或 append-only）
│
├── 元数据 ──▶ Lakekeeper（REST Catalog: namespace、表、快照，PG 存元数据）
└── 数据文件 ─▶ OSS 普通 Bucket（S3 兼容）
│
┌─────────────┼──────────────────┐
Spark/Trino   DuckDB（查询引擎）   StarRocks
│
▼ .pl() 零拷贝
Polars（结果容器 + 格式化输出）
```

> 注：存储是**普通 OSS Bucket + 自建 Lakekeeper**，不是阿里云 OSS Tables 托管服务。

---

## 2. 各组件职责

| 组件 | 角色 |
|---|---|
| MySQL | 数据源，binlog 供 CDC 消费 |
| RisingWave | 流式引擎：CDC 采集 + 存入前的清洗/转换 + 写 Iceberg |
| Lakekeeper | Iceberg REST Catalog，管元数据（存 PG），原子提交保证下游只看到完整快照 |
| OSS | S3 兼容对象存储，存数据文件和 metadata 文件 |
| DuckDB | 单机查询引擎：ATTACH REST Catalog → 读 Iceberg → 执行 SQL（真正干活） |
| Polars | **仅作结果容器**：`.pl()` 零拷贝接收 Arrow 结果，不做数据处理 |

**职责一句话**：RisingWave 负责流式处理和写入，Lakekeeper + OSS 负责存储管理，DuckDB 负责查询执行，Polars 只当零拷贝结果容器做格式化输出——每个组件只干一件事，职责零重叠。

---

## 3. 搭建步骤

### 3.1 MySQL 前置

```sql
SET GLOBAL binlog_format = ROW;
SET GLOBAL binlog_row_image = FULL;
```

CDC 用户权限：`SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT`

### 3.2 Lakekeeper

- Docker 启动，REST API `http://lakekeeper:8181/catalog`
- 建 Project + Warehouse，存储选 **S3 类型**（OSS endpoint / bucket / AK/SK）
- ⚠️ **必须注册 server 并分配到 Warehouse**，否则写入失败
- 认证：本地开发可关闭；生产用 token / credential vending

### 3.3 RisingWave: Source + Sink

```sql
CREATE SOURCE orders_src WITH (
    connector = 'mysql-cdc',
    hostname = 'mysql-host', port = 3306,
    username = 'cdc_user', password = 'secret',
    database.name = 'shop', table.name = 'orders',
    snapshot = true
);

CREATE SINK orders_daily AS
SELECT city, date_trunc('day', order_time) AS day,
       count(*) AS order_cnt, sum(amount) AS total_amount
FROM orders_src
GROUP BY city, day
WITH (
    connector = 'iceberg',
    type = 'append-only',  -- 能 append-only 就别 upsert
    catalog.type = 'rest',
    catalog.uri = 'http://lakekeeper:8181/catalog',
    warehouse = 'demo',
    database.name = 'shop_db', table.name = 'orders',
    s3.endpoint = 'https://oss-cn-hangzhou-internal.aliyuncs.com',
    s3.region = 'cn-hangzhou',
    s3.access.key.id = '<AK>', s3.secret.access.key = '<SK>'
);
```

- `type = 'upsert'`：必须有 `primary_key`，写 equality delete（Iceberg v2），适合会 UPDATE/DELETE 的表
- `type = 'append-only'`：文件布局好、查询性能优，优先选择

### 3.4 DuckDB + Polars 查询（CLI 场景）

```python
import duckdb
con = duckdb.connect()
con.execute("INSTALL iceberg; LOAD iceberg;")
con.execute("ATTACH 'lakekeeper' AS ice (TYPE ICEBERG, REST 'http://lakekeeper:8181/catalog')")
# OSS 凭证：
# SET s3_endpoint / s3_access_key_id / s3_secret_access_key / s3_url_style='vhost'

result = con.execute(duckdb_sql).pl()  # ★ 零拷贝：Arrow 直转 Polars

# 输出
result.height            # 行数（日志/耗时统计）
result.to_dicts()        # → json.dumps（--output-format json）
_df_to_markdown(result)  # → markdown 表（--output-format table）
```

**选 Polars 而非 pandas 的理由**：`.pl()` 走 Arrow 零拷贝无序列化开销；且全链路无数据处理逻辑，只遍历输出。

**Polars 的定位澄清**：Polars 是查询/分析侧工具，不参与流式链路的存入。存入前的处理在 RisingWave 完成（物化视图 / SQL / UDF）；Polars 只在存入后做分析，在本链路中仅当 DuckDB 的结果容器。

---

## 4. Polars / DuckDB 在链路中的角色

```
查询链路：DuckDB ATTACH Lakekeeper → 读 Iceberg → 执行 SQL（真正干活）
│
▼ .pl() 零拷贝（Arrow，无序列化开销）
pl.DataFrame
```

拿到 `pl.DataFrame` 后只有三件事：

| 操作 | 用途 |
|---|---|
| `result.height` | 行数统计（日志、耗时） |
| `result.to_dicts()` | 转 `list[dict]` 后 `json.dumps` 输出（`--output-format json`） |
| `_df_to_markdown(result)` | 转 markdown 表格（`--output-format table`） |

细节优化：

- `to_dicts()` 是输出链路里最贵的一步（Arrow 列式 → Python dict 行式）；大结果集可改走 `write_ndjson` / Arrow 原生序列化，或先 LIMIT
- markdown 输出可复用 Polars 默认 repr（竖线表格）
- 物化前想知道行数：`SELECT count(*) FROM (子查询)`

---

## 5. 关键设计决策

1. **存入前的处理全部在 RisingWave 完成**（物化视图/SQL），写入 Iceberg 的已是干净数据，下游直接查
2. **`append-only` 优先于 `upsert`**：upsert 产生 equality delete 文件，写放大 + 查询性能差
3. **DuckDB 为查询引擎，Polars 只做容器**：职责不重叠
4. **同地域 + OSS 内网 endpoint + https**：省流量费和延迟
5. **原子提交**：下游只看到完整快照，一致性由 Iceberg commit 保证
6. **存储形态确认为普通 OSS Bucket + Lakekeeper**（非 OSS Tables 托管）：Lakekeeper 必须保留，小文件治理自己负责

---

## 6. 常见坑清单

| 坑 | 对策 |
|---|---|
| OSS 报 S3 权限/找不到桶 | `s3_url_style='vhost'`（DuckDB）/ 正确的 pyiceberg `s3.endpoint` |
| Lakekeeper 写入失败 | 检查 server 是否已注册并分配到 Warehouse |
| upsert 小文件堆积 | Spark compaction procedure 或独立 compactor（普通 OSS 没有托管兜底） |
| schema 变更后查询异常 | 重建连接/重新 scan（按快照 schema 读） |
| `to_dicts()` 开销大 | 大结果集改走 `write_ndjson` / Arrow 原生序列化，或先 LIMIT |
| 认证 | 本地开发关 Lakekeeper 认证；生产用 token / credential vending |

---

## 7. 架构对比：asw vs search-todo

**核心差别**：asw 是"查的时候自己拉数据自己写"，search-todo 是"流式链路替你写好，你只管读"。

| 维度 | asw (search-workorder) | search-todo |
|---|---|---|
| Catalog | 阿里云 OSS Tables 托管服务（ARN + SigV4） | 自建 Lakekeeper（PG 存元数据） |
| 谁写入 | skill 自己的 Python 代码 | RisingWave CDC，skill 不写 |
| 同步时机 | Sync-on-Query：查询时调 ASW API，按上次 sync_time 增量拉，按 id Upsert | MySQL binlog 持续同步，秒级 |
| 数据新鲜度 | 取决于上次查询（两次查询之间是旧的） | 准实时 |
| 写方式 | PyIceberg append/overwrite（不支持 MoR 删除文件，更新=整表覆盖） | RisingWave upsert sink，merge-on-read，只写增量文件 |
| 读路径 | DuckDB iceberg 扩展连不上 OSS Tables（SigV4 signing-name 不兼容），只能 PyIceberg scan → Arrow → 注册进 DuckDB | 标准 REST catalog，DuckDB 直接 `ATTACH ... TYPE iceberg` |

### asw 的补偿性代码（全非业务逻辑）

```
OssTablesRestCatalog 子类化        → 修托管服务的协议非标（绕过坏掉的 /v1/config 端点）
双路凭证（client.* / s3.*）        → REST 层和数据层认证割裂
AWS_REQUEST_CHECKSUM_CALCULATION=when_required → 绕 OSS 不支持的 checksum trailer
Arrow schema 严格对齐              → required 字段非 nullable，否则静默失败
Sync-on-Query + 按 id Upsert       → 用 Python 模拟 CDC 语义
```

### search-todo 的读端

Lakekeeper 是标准 Iceberg REST 协议，DuckDB 原生支持，读端只有 30 行配置代码，上述坑全没有。

### 取舍总结

> asw：用 Python 在客户端手搓了一个不完整的流式同步引擎，来迁就托管服务的协议缺陷。
> search-todo：把同步、写入、目录三件事各交给专职组件，应用层只剩"读"。
> 前者的复杂度藏在代码里，后者的复杂度摆在 docker-compose 里（RisingWave + Lakekeeper + PG 三个容器）——后者的可维护性高一个量级。

**本质**：search-todo 不是没有运维成本，而是把成本从"代码复杂度"换成了"组件运维"。前者在每次协议升级/边界 case 时都要人肉补丁，后者是声明式配置一次到位。对数据量会增长、多用户查询的系统，这笔交换是划算的。

---

## 附：适用场景

- ✅ **CDC 实时入湖 + 单机轻量分析**：MySQL 变更分钟级落 Iceberg，DuckDB/Polars 跑报表、对账、数据验证
- ❌ **不适合**：数据量大到需要分布式查询（用 Spark/Trino）、或需要直接写 Iceberg（写入交给 RisingWave/Spark）
