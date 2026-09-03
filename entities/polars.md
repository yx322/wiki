---
title: Polars
type: entity
tags: [polars, data-processing, analytics]
created: 2026-07-21
updated: 2026-07-21
sources:
  - raw/articles/DeltaLake_DuckDB_Polars_Guide.md
  - raw/articles/polars_write_iceberg_oss_tables.md
confidence: high
contested: false
---

# Polars

高性能 DataFrame 库，底层用 Rust 编写，替代 Pandas 的"电动超跑"。

## 为什么这么快

### 1. Rust 语言重写

底层用追求极致性能的 Rust 编写，接近硬件极限。

### 2. 真正的多核并行

- Pandas：单线程（1 个核干活，其他围观）
- Polars：瞬间占满所有 CPU 核心，拆分任务同时计算

### 3. 懒惰模式（Lazy Mode）

输入命令后不立刻干活，先看一遍进行优化：

```
读 10G 文件 → 过滤北京 → 只看姓名
```

- **Pandas（笨办法）**：全读入内存 → 过滤 → 删列 → 内存爆炸
- **Polars（聪明办法）**：读文件时只读"姓名"列，且只捞"北京"的数据 → 内存极省

## 与 [[duckdb]] 的对比

| 维度 | DuckDB | Polars |
|------|--------|--------|
| 交互方式 | SQL 语句 | Python 链式调用 |
| 最擅长 | 跨文件查询、BI 报表 | 数据清洗、特征工程、矩阵运算 |
| 比喻 | 自动售货机 | 开放式厨房 |
| 控制力 | 黑盒计算 | 你指挥切菜、下锅、装盘 |

两者底层都遵循 Apache Arrow 标准，数据传输零成本，是最佳搭档。

## write_iceberg 限制

Polars 1.41.2 的 `write_iceberg()` 签名是 `(target, mode)`，不支持 `catalog_options` 参数。写入阿里云 OSS Tables 时遇到协议不兼容问题，详见 [[polars-iceberg-oss-tables]]。

## 在湖仓架构中的角色

```
[[delta-lake]] / [[iceberg]]  →  [[duckdb]]  →  [[polars]]
   存数据在云端                    快速吸出数据         清洗和 ML 准备
```

## 参见

- [[duckdb]] — 分析搭档
- [[lakehouse]] — 湖仓一体架构
- [[iceberg]] — 表格式规范
