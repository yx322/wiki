---
title: 数据管道工作流 (DuckDB + Delta Lake + Polars)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [duckdb, delta-lake, polars, data-pipeline]
sources: [raw/articles/data-pipeline-workflow.md]
confidence: high
---

# 数据管道工作流

config.py + query.py + sync.py + auth.py 四个核心模块的组合使用流程。

## 三条主线

```
config.py (配置中心) → YAML + 环境变量 → pydantic-settings → cfg 对象
       ↓                    ↓
   query.py             sync.py
   (查询引擎)           (数据同步)
   DuckDB + delta       httpx → Polars → Delta
   执行 SQL             拉取 → 清洗 → 写入
       ↓                    ↓
       └── Sync-on-Query ──┘
              ↓
       S3 / MinIO / Delta Lake (持久化存储)
```

## 查询流程

用户自然语言提问 → AI 生成 SQL → query.py 执行：

1. **路径渲染**：`{vehicle}` → `s3://vehicle/vehicle_analysis/vehicle_records`
2. **数据新鲜度检查**：`SELECT MAX(sync_time)` 对比阈值，过期则后台触发 sync.py
3. **只读校验**：解析 SQL AST，拦截 INSERT/UPDATE/DELETE
4. **SQL 注入**：`delta_scan('{vehicle}')` → `delta_scan('s3://...')`
5. **DuckDB 连接**：加载 delta + httpfs 扩展，设置 S3 凭证
6. **执行**：`con.execute(sql).pl()` 返回 Polars DataFrame
7. **输出**：JSON 或 Markdown 表格

## 同步流程

API → Polars → Delta Lake：

1. **获取认证**：从环境变量读 `CONTEXT_METADATA_ACCESS_TOKEN`
2. **增量起点**：`SELECT MAX(add_time)` + 1 秒
3. **API 拉取**：httpx 分页循环，每页 sleep(0.3) 防限流
4. **Polars ETL**：LazyFrame → 选择列 → 类型转换 → 文本映射 → 注入 sync_time
5. **写入前去重**：DuckDB register + LEFT JOIN 过滤已存在 id
6. **写入 Delta**：`write_deltalake(mode="append")`

## Sync-on-Query

查询时自动触发同步（不阻塞）：

```python
subprocess.Popen(
    [sys.executable, "sync.py", "sync", "--auto"],
    start_new_session=True,  # 脱离父进程
    stdin=DEVNULL, stdout=DEVNULL, stderr=DEVNULL
)
```

- 查询不阻塞等待同步完成
- 同步完成后，下次查询自动读取新数据

## 参见
- [[duckdb]] — 查询引擎
- [[delta-lake]] — 存储格式
- [[polars]] — ETL 引擎
- [[lakehouse]] — 湖仓一体

^[raw/articles/data-pipeline-workflow.md]
