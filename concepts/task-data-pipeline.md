---
title: OA 待办数据管道 (CDC → RisingWave → Iceberg)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [risingwave, iceberg, cdc, data-pipeline, lakehouse]
sources: [raw/articles/task-data-pipeline.md]
confidence: high
---

# OA 待办数据管道

MySQL CDC → RisingWave → Lakekeeper Iceberg (OSS) 数据写入流水线 + NL-to-SQL 查询技能。

## 整体架构

```
MySQL (oa_task)  →  RisingWave (CDC)  →  物化视图  →  Iceberg Sink  →  OSS
                                                                    ↓
                                                         skills/task 查询层
                                                         AI → SQL → DuckDB → JSON
```

## 职责划分

| 模块 | 职责 |
|------|------|
| 基础设施 (risingwave/) | Docker 编排 RisingWave + Lakekeeper + PostgreSQL |
| CDC 写入任务 | MySQL 增量同步、物化视图、Iceberg Sink |
| 查询技能 (skills/task/) | NL-to-SQL、只读查询 Iceberg 表、返回业务结果 |

**核心原则**：写入与查询分离。RisingWave 持续负责数据新鲜度；查询技能仅读取，不调用 OA API。

## 双表分区策略

| 表名 | 分区字段 | 优化场景 |
|------|----------|---------|
| task_records | task_empId | 我的待办、待我处理、我完成的、待验收 |
| task_records_by_creator | create_empId | 我发起的 |

## 技术栈

| 层级 | 技术 |
|------|------|
| 源库 | MySQL（binlog ROW 格式） |
| 流引擎 | RisingWave |
| Catalog | Lakekeeper（REST Catalog，元数据存 PostgreSQL 16） |
| 对象存储 | 阿里云 OSS |
| 表格式 | Apache Iceberg v2（upsert + merge-on-read） |
| 查询引擎 | DuckDB + PyIceberg + Polars |
| Agent 框架 | SkillForge |

## CDC 流水线步骤

| 步骤 | SQL 文件 | 动作 |
|------|----------|------|
| 1 | 00_create_mysql_cdc_table | 创建 MySQL CDC 源表 |
| 2 | 01_create_materialized_view | 创建物化视图 mv_oa_task |
| 3 | 02_create_lakekeeper_connection | 创建 Iceberg REST 连接 |
| 4 | 03_create_iceberg_sink | Sink → task_records |
| 5 | 04_create_iceberg_sink_by_creator | Sink → task_records_by_creator |

## 查询层 6.2 执行流程

1. **用户身份注入**：从上下文 token 解析 current_user_id
2. **只读校验**：拦截写操作
3. **分区路由**：WHERE 含 create_empId → 读 by_creator 表；否则读主表
4. **谓词下推**：简单 AND 等值条件转为 PyIceberg row_filter
5. **列裁剪**：从 SELECT/WHERE 提取所需列
6. **DuckDB 执行**：Arrow Table 注册为虚拟表，执行 SQL
7. **输出**：stdout JSON，进度日志 stderr

## 意图 → SQL 映射

| 用户说法 | SQL 条件 |
|---------|---------|
| 我的待办 | task_empId = {current_user_id} |
| 待我处理 | task_empId = {current_user_id} AND task_status = 0 |
| 我发起的 | create_empId = {current_user_id} |
| 我完成的 | task_empId = {current_user_id} AND task_status IN (1, 2) |
| 待验收 | check_emp_id = {current_user_id} AND task_status IN (1, 2) |

## 参见
- [[risingwave]] — 流数据库
- [[iceberg]] — 表格式
- [[cdc-vs-api-sync]] — CDC vs API 同步
- [[skillforge]] — Agent 运行时
- [[lakekeeper]] — 自建 REST Catalog（元数据层）
- [[sync-on-query-vs-cdc]] — asw vs search-todo 架构对比（本链路为 search-todo 侧）

^[raw/articles/task-data-pipeline.md]
