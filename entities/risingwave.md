---
title: RisingWave
type: entity
tags: [risingwave, streaming, lakehouse, cdc]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/湖仓一体化（纯接口模式）性能重构落地方案.md
  - raw/articles/湖仓一体化性能重构落地方案.md
confidence: high
contested: false
---

# RisingWave

流数据库，持续维护物化视图，SQL 声明式定义，数据库自动管理增量计算和状态。

## 核心定位

RisingWave 不是传统 OLAP（不接受用户交互查询），而是**持续维护物化视图**。本质是牺牲灵活性，换取性能、延迟和并发 — 在数据写入时实时计算聚合结果，查询时直接读预计算表，延迟从分钟级降到毫秒级。

## 物化视图原理

RisingWave 的物化视图采用"常驻内存、实时滚动演进（IVM）"技术：

```
CREATE MATERIALIZED VIEW 的瞬间
  ↓
流水线构建完毕，24 小时常驻内存运行
  ↓
新数据流入时，只拿新数据的 key 去内存哈希表比对
  ↓
时间更新 → 内存里擦除旧状态、接纳新状态（2~3 毫秒）
  ↓
下游查询直接白嫖内存里已经就绪的最新去重结果（10 毫秒）
```

**关键**：不是访问一次构建一次，而是 24 小时常驻内存、实时滚动演进。

## 两种接入模式

### 模式 A：MySQL CDC（终极方案）

```
MySQL binlog  →  RisingWave CDC 引擎  →  基础表（天生去重）  →  Iceberg Sink
  毫秒级无感捕捉         自动 Upsert           0 代码维护         每 2 分钟落湖
```

**优势**：
- 完全消灭接口与 Python 代码
- binlog 带 INSERT/UPDATE 信号，RisingWave 自动 Upsert
- 基础表天生就是最新、最干净、完全去重的最终态

### 模式 B：Python 拉取接口（过渡方案）

```
OA 接口  →  Python 定时拉取  →  RisingWave 基础表  →  物化视图（去重）  →  Iceberg Sink
                                    20 毫秒写入         ROW_NUMBER()        每 2 分钟落湖
```

**需要额外建去重视图**：

```sql
CREATE MATERIALIZED VIEW mv_clean AS
SELECT * EXCLUDE (rn) FROM (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY task_id ORDER BY sync_time DESC) as rn
    FROM source_table
) WHERE rn = 1;
```

## Iceberg Sink 配置

```sql
CREATE SINK iceberg_sink_to_oss 
FROM mv_clean
WITH (
    connector = 'iceberg',
    type = 'upsert',
    catalog.type = 'rest',
    catalog.uri = 'https://aliyuncs.com',
    warehouse = 'oss://bucket/warehouse/',
    sink.flush.interval = '120s',
    sink.checkpoint.interval = '60s'
);
```

**关键参数**：
- `type = 'upsert'`：覆盖更新语义
- `sink.flush.interval = '120s'`：每 2 分钟大批次向 OSS 写入
- `sink.checkpoint.interval = '60s'`：每 1 分钟对齐元数据，物理斩断 Commit 冲突

## 性能提升

| 指标 | PyIceberg 直写 | RisingWave + Sink |
|------|---------------|-------------------|
| 写入延迟 | 45 秒 | 20 毫秒 |
| 查询延迟 | 8.6 秒 | 10 毫秒 |
| Commit 冲突 | 频繁 | 消灭 |
| OOM 风险 | 高 | 无 |

## 参见

- [[iceberg]] — 落湖格式
- [[lakehouse]] — 湖仓一体架构
- [[cdc-vs-api-sync]] — CDC 与接口同步对比
