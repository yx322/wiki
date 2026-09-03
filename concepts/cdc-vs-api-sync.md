---
title: CDC vs API 同步
type: concept
tags: [cdc, api-sync, risingwave, streaming]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/湖仓一体化性能重构落地方案.md
confidence: high
contested: false
---

# CDC vs API 同步

两种数据同步模式的对比：CDC（Change Data Capture）监听数据库物理日志 vs API 定时拉取接口。

## 模式 A：MySQL CDC（终极方案）

```
MySQL binlog  →  RisingWave CDC 引擎  →  基础表（天生去重）  →  Iceberg Sink
  毫秒级无感捕捉         自动 Upsert           0 代码维护         每 2 分钟落湖
```

### 优势

- **完全消灭接口与 Python 代码**：接口直接退役，零代码维护
- **binlog 带 INSERT/UPDATE 信号**：RisingWave 自动 Upsert，基础表天生就是最新、最干净、完全去重的最终态
- **不需要去重视图**：数据在流进 RisingWave 的那一瞬间，就已经处于最新状态

### 前置准备

MySQL 端配置：

```sql
-- 确认 binlog 已开启
SHOW VARIABLES LIKE 'log_bin';          -- 必须是 ON
SHOW VARIABLES LIKE 'binlog_format';    -- 必须是 ROW

-- 创建 CDC 只读账号
CREATE USER 'rw_cdc_user'@'%' IDENTIFIED BY 'your_password';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'rw_cdc_user'@'%';
FLUSH PRIVILEGES;
```

## 模式 B：Python 拉取接口（过渡方案）

```
OA 接口  →  Python 定时拉取  →  RisingWave 基础表  →  物化视图（去重）  →  Iceberg Sink
                                    20 毫秒写入         ROW_NUMBER()        每 2 分钟落湖
```

### 劣势

- **需要维护 Python 代码**：定时调度、接口调用、错误处理
- **接口限流和超时**：网络不稳定，接口可能改版
- **需要额外建去重视图**：Python 盲插会引入时间重复行，需要 `ROW_NUMBER()` 去重

### 去重视图

```sql
CREATE MATERIALIZED VIEW mv_clean AS
SELECT * EXCLUDE (rn) FROM (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY task_id ORDER BY sync_time DESC) as rn
    FROM source_table
) WHERE rn = 1;
```

## 对比总结

| 维度 | CDC | API 同步 |
|------|-----|---------|
| 代码维护 | 0 行 | 需要维护 Python |
| 接口依赖 | 无 | 依赖接口稳定性 |
| 去重逻辑 | 自动 Upsert | 需要物化视图 |
| 延迟 | 毫秒级 | 分钟级（定时拉取） |
| 前置条件 | 需要 binlog 权限 | 只需要接口 |

## 决策建议

- **能用 CDC 就用 CDC**：完全消灭接口和 Python 代码，零维护
- **只能用接口时用 API 同步**：过渡方案，需要额外建去重视图

## 参见

- [[risingwave]] — 流数据库
- [[iceberg]] — 落湖格式
- [[lakehouse]] — 湖仓一体架构
