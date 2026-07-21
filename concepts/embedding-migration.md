---
title: 嵌入模型迁移 (Ollama → DashScope)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedding-migration, dashscope, migration, surrealdb]
sources: [raw/articles/SurrealDB_DashScope_Migration_Guide.md, raw/articles/SurrealDB向量集成方案分析.md]
confidence: high
---

# 嵌入模型迁移 (Ollama → DashScope)

将向量生成方案从本地 Ollama bge-m3 迁移到云端 [[dashscope]] text-embedding-v4 的完整流程。

## 核心原则

**不同模型的向量不可混用**。Ollama bge-m3 和 DashScope text-embedding-v4 生成的向量在完全不同的坐标空间，余弦相似度计算毫无意义。

## 迁移策略：双向量并行

保留旧字段做备份，新增字段给业务用：

```
旧字段：embedding (Ollama bge-m3，保留)
新字段：embedding_dashscope (DashScope v4，业务使用)
```

## 步骤

### 1. SurrealDB 端

```sql
-- 新增字段
DEFINE FIELD embedding_dashscope ON document_chunks TYPE array<float>;

-- 定义嵌入函数
DEFINE FUNCTION fn::dashscope_embed($text: string) { ... };

-- 批量刷新（WHILE 循环 + TRY/CATCH + SLEEP 限流）
```

### 2. Python 端（knowledge_base 项目）

| 文件 | 变更 |
|------|------|
| config.yaml | 新增 dashscope 配置 |
| settings.py | 新增 DashScopeSettings 类 |
| retrieval_client.py | SQL 改为 fn::dashscope_embed，字段改 embedding_dashscope |
| vectorize.py | 请求目标改 DashScope URL |
| store.py | 索引改 embedding_dashscope HNSW DIMENSION 1024 |
| document.py / webhook.py | 向量化改调 DashScope API |

### 3. 索引重建

```python
DEFINE INDEX vector_idx_dashscope ON {table} FIELDS embedding_dashscope HNSW DIMENSION 1024 DIST COSINE EFC 100
```

## 数据清洗方案

- **方案 A（推荐）**：清空重跑 — `DELETE FROM table;` 再跑导入脚本
- **方案 B**：原地刷新 — WHILE 循环调用 fn::dashscope_embed 覆盖旧向量

## 注意事项

- 迁移前备份：`CREATE backup AS SELECT * FROM table;`
- 批量处理用 LIMIT 分批，避免长事务回滚
- SLEEP 0.1 避免 DashScope API 限流
- 确认稳定后可清理旧字段释放空间

## 参见
- [[dashscope]] — 新嵌入模型
- [[ollama-vs-dashscope]] — 方案对比
- [[surrealdb]] — 数据库集成

^[raw/articles/SurrealDB_DashScope_Migration_Guide.md]
^[raw/articles/SurrealDB向量集成方案分析.md]
