---
title: HNSW 向量索引
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [hnsw-index, vector-search, surrealdb, performance]
sources: [raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md, raw/articles/SurrealDB索引定义解析.md, raw/articles/SurrealDB电商混合搜索与多维推荐方案.md]
confidence: high
---

# HNSW 向量索引

Hierarchical Navigable Small World — 高效近似最近邻（ANN）搜索算法，将向量搜索复杂度从 O(N) 降至对数级别。

## SurrealDB 中的定义

```sql
DEFINE INDEX idx ON table FIELDS embedding HNSW DIMENSION 1024 DIST COSINE EFC 100 CONCURRENTLY;
```

### 参数说明

| 参数 | 含义 | 调优建议 |
|------|------|---------|
| DIMENSION | 向量维度 | 必须与嵌入模型一致（text-embedding-v4 = 1024） |
| DIST | 距离函数 | COSINE（归一化向量）、EUCLIDEAN |
| EFC | 查询扩展因子 | 100~200 更准，50~80 更快 |
| CONCURRENTLY | 异步构建 | 不阻塞写入，生产环境必加 |

## 搜索语法

```sql
-- 基本 ANN 搜索：返回 N 个结果，候选池 ef
WHERE embedding <|N,ef|> $query_vector

-- 带距离函数
WHERE embedding <|10,COSINE|> $qvec

-- 取分数
SELECT (1.0 / (1.0 + vector::distance::knn())) AS score
```

## 与全文搜索的配合

HNSW 索引负责语义召回，与 [[bm25]] 全文索引互补：
- HNSW 捕捉语义意图（"推荐安全帽" ≈ "安全防护帽"）
- BM25 捕捉精确关键词（型号、编码等）
- 两路结果通过 [[rrf-fusion]] 或 [[search-linear]] 合并

## 参见
- [[two-stage-search-pipeline]] — 两阶段流水线工业级完整实现
- [[surrealdb]] — 数据库
- [[hybrid-search]] — 混合搜索架构
- [[dashscope]] — 嵌入模型

^[raw/articles/SurrealDB索引定义解析.md]
^[raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md]
