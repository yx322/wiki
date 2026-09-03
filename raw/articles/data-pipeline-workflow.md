# 工具组合使用流程 — 数据架构全景

> 本文档从数据流向角度，完整梳理 config.py、query.py、sync.py、auth.py 四个核心模块的组合使用流程。

## 整体架构：三条主线

```
┌─────────────────────────────────────────┐
│ config.py (配置中心)                     │
│ YAML + 环境变量 → pydantic-settings      │
│ 输出: cfg 对象（全局共享）                │
└──────┬──────────────┬──────────────────┘
       │              │
┌──────▼─────┐  ┌─────▼────────────────┐
│ query.py   │  │ sync.py              │
│ (查询引擎) │  │ (数据同步)           │
│            │  │                      │
│ DuckDB     │  │ httpx → Polars       │
│ + delta    │  │ → Delta              │
│ 执行 SQL   │  │ 拉取 → 清洗 → 写入   │
└──────┬─────┘  └─────┬────────────────┘
       │              │
       │  Sync-on-Query│ 后台子进程
       │  (Popen)     │
       └──────────────┘
              │
       ┌──────▼────────┐
       │ S3 / MinIO    │
       │ Delta Lake    │
       │ (持久化存储)   │
       └───────────────┘
```

---

## 流程 1：查询流程（用户提问 → 返回结果）

这是最常用的路径。用户用自然语言提问，AI 生成 SQL，`query.py` 执行并返回。

```
用户: "今天有多少车辆出厂？"
│
▼
┌─── AI 大脑 ──────────────────────────────────────────────────────┐
│                                                                 │
│ 1. 阅读 SKILL.md 中的 schema（第 45-84 行）                       │
│    → 知道 vehicle_records 表有 device_type、add_time 等字段       │
│                                                                 │
│ 2. 根据 SQL 模板（第 108-160 行）生成 DuckDB SQL：                │
│    SELECT device_type_text, COUNT(*) as cnt                     │
│    FROM delta_scan('{vehicle}')                                 │
│    WHERE DATE(add_time) = CURRENT_DATE AND device_type = 2      │
│    GROUP BY device_type_text                                    │
│                                                                 │
│ 3. 调用 get_skill_script 执行：                                  │
│    python scripts/query.py run --sql "上述 SQL"                  │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─── query.py: run() [第 288 行] ─────────────────────────────────┐
│                                                                 │
│ ┌─ Step 1: _render_paths() [第 79 行]                           │
│ │    {vehicle} → "s3://vehicle/vehicle_analysis/vehicle_records"│
│ │    {approval} → "s3://vehicle/vehicle_analysis/approval_records"│
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 2: ensure_data_freshness() [第 186 行]                  │
│ │                                                               │
│ │ ├─ get_last_sync_time() [第 166 行]                           │
│ │ │    └─ DuckDB: SELECT MAX(sync_time) FROM delta_scan(...)    │
│ │ │                                                            │
│ │ ├─ _table_sync_is_stale() [第 170 行]                         │
│ │ │    └─ now - last_sync > threshold?                         │
│ │ │                                                            │
│ │ └─ (如果过期) _trigger_sync() [第 227 行]                     │
│ │       └─ subprocess.Popen([sync.py, sync, --auto])           │
│ │          start_new_session=True → 后台运行，父进程不等待       │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 3: _check_readonly(sql) [第 260 行]                     │
│ │    └─ duckdb.extract_statements(sql)                          │
│ │       → 解析 AST，检查首 token 是否为 INSERT/UPDATE/DELETE 等  │
│ │       → 是写操作 → raise_exit(BUSINESS_ERROR)                 │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 4: _inject_sql(sql) [第 88 行]                          │
│ │    └─ delta_scan('{vehicle}') → delta_scan('s3://.../...')    │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 5: get_duckdb_connection() [第 98 行]                   │
│ │                                                               │
│ │ ├─ os.environ["AWS_ACCESS_KEY_ID"] = cfg.s3.access_key_id    │
│ │ │   os.environ["AWS_SECRET_ACCESS_KEY"] = ...                 │
│ │ │   → Delta Kernel (Rust) 读环境变量                          │
│ │ │                                                            │
│ │ ├─ duckdb.connect() → con                                     │
│ │ │                                                            │
│ │ ├─ con.execute("CREATE SECRET s3_vehicle (...)")              │
│ │ │   → DuckDB 原生 S3 凭证                                     │
│ │ │                                                            │
│ │ ├─ con.execute("INSTALL delta; LOAD delta;")                  │
│ │ │   con.execute("INSTALL httpfs; LOAD httpfs;")               │
│ │ │   → 加载 S3 + Delta 扩展                                    │
│ │ │                                                            │
│ │ └─ yield con → 调用方使用                                     │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 6: con.execute(execution_sql).pl() [第 316 行]          │
│ │                                                               │
│ │ ├─ DuckDB 解析 SQL                                            │
│ │ ├─ 通过 httpfs 扩展访问 S3                                    │
│ │ ├─ 通过 delta 扩展读取 _delta_log + parquet 文件              │
│ │ ├─ 执行过滤、聚合、JOIN                                       │
│ │ └─ 返回 Polars DataFrame                                      │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 7: 输出 [第 325-328 行]                                 │
│ │                                                               │
│ │ ├─ --output-format json:                                     │
│ │ │    typer.echo(json.dumps(df.to_dicts())) → stdout          │
│ │ │                                                            │
│ │ └─ --output-format table:                                    │
│ │      typer.echo(_df_to_markdown(df)) → stdout                │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
│
▼
AI 收到 JSON → 整理为自然语言回答用户
```

### 工具协作关系

| 工具 | 职责 |
|------|------|
| `pydantic-settings` | 提供配置 → `cfg` 全局对象 |
| `typer` | 接收命令行参数 → 路由到 `run()` 函数 |
| `duckdb` | 解析 SQL + 连接 S3 + 执行查询 |
| `structlog` | 记录每个步骤的结构化日志（stderr） |
| `subprocess` | 触发后台同步（如果需要） |
| `json` | 序列化结果到 stdout |

---

## 流程 2：同步流程（API → Polars → Delta Lake）

这是数据入库的路径。从两个 API 拉取原始数据，清洗后写入 S3。

```
┌─── sync.py: sync() [第 834 行] ─────────────────────────────────┐
│                                                                 │
│ ┌─ Step 0: 获取认证 [第 845 行]                                 │
│ │    └─ auth.py: get_access_token_from_env()                    │
│ │       → os.environ.get("CONTEXT_METADATA_ACCESS_TOKEN")       │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 1: 增量起点计算 [第 850-862 行]                         │
│ │                                                               │
│ │ ├─ get_last_sync_time() [第 74 行]                            │
│ │ │    └─ DuckDB: SELECT MAX(add_time) FROM delta_scan(...)     │
│ │ │                                                            │
│ │ └─ next_add_time_start() [第 101 行]                          │
│ │       └─ last_time + 1 秒 → add_time_start                   │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 2: API 拉取 [第 874 行]                                 │
│ │    └─ fetch_all_vehicle_records(access_token, add_time_start) │
│ │       [第 283 行]                                             │
│ │                                                               │
│ │ ├─ auth.py: BackendApiClient(...)                             │
│ │ │    → httpx.Client(timeout=30)                               │
│ │ │    → client.get(url, headers={"oa-access-token": ...})      │
│ │ │    → 分页循环，每页 sleep(0.3) 防限流                        │
│ │ │                                                            │
│ │ └─ 返回 List[dict] 原始数据                                   │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 3: Polars ETL [第 882 行]                               │
│ │    └─ etl_vehicle_records(raw_vehicles) [第 499 行]           │
│ │                                                               │
│ │ ├─ pl.LazyFrame(raw_records)                                  │
│ │ ├─ 选择目标列（12 列）                                        │
│ │ ├─ str.to_datetime("add_time") → 时间解析                     │
│ │ ├─ cast(Int64, strict=False) → 类型强转                       │
│ │ ├─ replace_strict({1:"入口", 2:"出口"}) → 文本映射             │
│ │ ├─ pl.lit(datetime.now()) → 注入 sync_time                    │
│ │ └─ lf.collect() → DataFrame                                   │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 4: 写入前去重 [第 883 行]                               │
│ │    └─ filter_new_vehicle_records(df, table_path) [第 237 行]  │
│ │                                                               │
│ │ ├─ DuckDB: con.register("staging", df.to_arrow())             │
│ │ │                                                            │
│ │ ├─ SQL:                                                      │
│ │ │    SELECT s.* FROM staging s                                │
│ │ │    LEFT JOIN delta_scan('{table_path}') e                   │
│ │ │    ON s.id = e.id                                           │
│ │ │    WHERE e.id IS NULL                                       │
│ │ │                                                            │
│ │ └─ 返回新记录（已存在的 id 被过滤）                            │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 5: 写入 Delta Lake [第 884 行]                          │
│ │    └─ write_to_delta(df, table_path, "vehicle_records")       │
│ │       [第 783 行]                                             │
│ │                                                               │
│ │ ├─ _get_storage_options() [第 673 行]                         │
│ │ │    → {"AWS_ACCESS_KEY_ID": ...,                             │
│ │ │       "AWS_SECRET_ACCESS_KEY": ...,                         │
│ │ │       "AWS_ENDPOINT_URL": "http://api.minio.s",             │
│ │ │       "allow_http": "true"}                                 │
│ │ │                                                            │
│ │ ├─ try: DeltaTable(table_path, storage_options)               │
│ │ │    → 表存在 → write_deltalake(mode="append")                │
│ │ │                                                            │
│ │ └─ except TableNotFoundError:                                 │
│ │       → 表不存在 → write_deltalake(mode="overwrite")          │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─ Step 6: 审批同步（同上流程，略）[第 887-906 行]              │
│ │    └─ resolve_leave_date_range() → fetch → etl → 去重 → 写入   │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 工具协作关系

| 工具 | 职责 |
|------|------|
| `httpx` | 通过 `auth.py` 的 `BackendApiClient` 发起 API 请求 |
| `Polars` | 做 ETL：LazyFrame 过滤 → 类型转换 → collect 输出 |
| `DuckDB` | 辅助去重：`register("staging", df.to_arrow())` + LEFT JOIN |
| `deltalake` | 写入 S3：`write_deltalake()` + `DeltaTable()` 版本管理 |
| `structlog` | 记录每个阶段的结构化日志 |

---

## 流程 3：Sync-on-Query（查询时自动触发同步）

这是两条主线的交集，也是这个 skill 最巧妙的设计。

```
query.py: run()
│
├─ ensure_data_freshness() [第 186 行]
│  │
│  ├─ DuckDB: SELECT MAX(sync_time) FROM delta_scan(...)
│  │    → 返回 last_sync_time
│  │
│  ├─ 比较: now - last_sync > threshold?
│  │    → threshold = cfg.sync.auto_sync_threshold_hours * 3600
│  │    → 当前配置为 0，即每次查询都检查
│  │
│  └─ 如果过期:
│     │
│     ├─ _progress("数据已过期，正在自动同步...") → stderr
│     │
│     └─ _trigger_sync() [第 227 行]
│        │
│        ├─ cmd = [sys.executable, "sync.py", "sync", "--auto"]
│        │
│        ├─ subprocess.Popen(
│        │     cmd,
│        │     start_new_session=True,  # 脱离父进程
│        │     stdin=DEVNULL,           # 不阻塞
│        │     stdout=DEVNULL,          # 不阻塞
│        │     stderr=DEVNULL,          # 不阻塞
│        │     close_fds=True           # 关闭继承的文件描述符
│        │  )
│        │
│        └─ 父进程立即继续，不等待子进程完成
│
└─ 继续执行查询（可能读到旧数据，但下次查询就是新的了）
```

### 关键设计点

- **查询不阻塞等待同步完成**：同步在后台独立运行
- **自动生效**：同步完成后，下次查询自动读取新数据
- **适用场景**：适合"查询频繁、数据更新不频繁"的场景

---

## 工具协作全景图

```
┌─────────────────────────────────────────────────────────────────┐
│ config.py                                                       │
│ pydantic-settings 加载 YAML + 环境变量 → cfg 对象               │
│ 被所有脚本导入: from scripts.config import cfg                  │
└──────┬──────────────┬──────────────┬────────────────────────────┘
       │              │              │
       ▼              ▼              ▼
┌────────────┐  ┌────────────┐  ┌────────────┐
│ query.py   │  │ sync.py    │  │ auth.py    │
│            │  │            │  │            │
│ typer      │  │ typer      │  │ httpx      │
│ duckdb     │  │ httpx      │  │ structlog  │
│ structlog  │  │ polars     │  │            │
│ subprocess │  │ deltalake  │  │            │
│ json       │  │ duckdb     │  │            │
│            │  │ structlog  │  │            │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      │               │               │
      │               │               │
      ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│ S3 / MinIO (Delta Lake)                                         │
│                                                                 │
│ vehicle_records/                                                │
│   _delta_log/                                                   │
│   *.parquet                                                     │
│                                                                 │
│ approval_records/                                               │
│   _delta_log/                                                   │
│   *.parquet                                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 数据流向总结

| 方向 | 工具链 | 说明 |
|------|--------|------|
| API → S3 | `httpx` → `polars` → `deltalake` | 拉取 → 清洗 → 写入 |
| S3 → 用户 | `duckdb` → `json` | 查询 → 序列化 → stdout |
| 配置 → 所有脚本 | `pydantic-settings` → `cfg` | 全局共享配置 |
| 日志 → 终端/文件 | `structlog` | 结构化日志输出 |
| 查询触发同步 | `subprocess` | 后台启动 `sync.py` |

---

## 核心文件索引

| 文件 | 关键函数/行号 | 职责 |
|------|---------------|------|
| `config.py` | - | 配置中心，pydantic-settings 加载 |
| `query.py` | `run()` 第 288 行 | 查询入口 |
| `query.py` | `_render_paths()` 第 79 行 | 路径模板渲染 |
| `query.py` | `ensure_data_freshness()` 第 186 行 | 数据新鲜度检查 |
| `query.py` | `_trigger_sync()` 第 227 行 | 后台触发同步 |
| `query.py` | `_check_readonly()` 第 260 行 | SQL 只读校验 |
| `query.py` | `get_duckdb_connection()` 第 98 行 | DuckDB 连接管理 |
| `sync.py` | `sync()` 第 834 行 | 同步入口 |
| `sync.py` | `fetch_all_vehicle_records()` 第 283 行 | API 数据拉取 |
| `sync.py` | `etl_vehicle_records()` 第 499 行 | Polars ETL 清洗 |
| `sync.py` | `filter_new_vehicle_records()` 第 237 行 | 写入前去重 |
| `sync.py` | `write_to_delta()` 第 783 行 | Delta Lake 写入 |
| `auth.py` | `get_access_token_from_env()` | 认证令牌获取 |
| `auth.py` | `BackendApiClient` | HTTP 客户端封装 |
