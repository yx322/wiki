---
title: SurrealDB
created: 2026-07-21
updated: 2026-07-21
type: entity
tags: [surrealdb, vector-search, fulltext-search, architecture]
sources: [raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md, raw/articles/SurrealDB向量集成方案分析.md, raw/articles/SurrealDB索引定义解析.md, raw/articles/SurrealDB电商混合搜索与多维推荐方案.md]
confidence: high
---

# SurrealDB

多模型数据库，同时支持文档存储、图数据库、向量搜索和全文检索。在本项目中作为商品搜索和知识库的核心存储引擎。

## 核心能力

### 向量搜索
- 支持 [[hnsw-index]] 索引，语法：`embedding <|N,ef|> $query_vector`
- 距离函数：COSINE、EUCLIDEAN
- 维度需与嵌入模型一致（DashScope text-embedding-v4 = 1024 维）

### 全文搜索
- [[bm25]] 评分算法
- 自定义分析器（如 [[ngram-analyzer]]）
- 搜索语法：`content @0@ $keyword`，配合 `search::score(0)` 取分
- 支持高亮（HIGHLIGHTS）

### 混合搜索融合
- [[rrf-fusion]]：`search::rrf([$vs, $ft], limit, k)` — 倒数排名融合，只看排名不看分数
- [[search-linear]]：`search::linear([[$set, 'score_field'], ...], weights, limit, 'minmax')` — 线性加权融合，支持 Min-Max 归一化

### 内置函数
- `vector::similarity::cosine()` — 余弦相似度
- `vector::distance::knn()` — KNN 距离
- `math::log1p()` — 对数平滑
- `http::post()` — 在数据库内发起 HTTP 请求（用于调用 [[dashscope]] API）
- `DEFINE FUNCTION fn::xxx()` — 自定义函数

## 在本项目中的使用

- **知识库系统**：`document_chunks` 表存储文档切片 + 向量
- **电商搜索**：`goods` / `goods_test` 表存储商品 + 向量 + 全文索引
- **报价系统**：商品搜索集成在 entrance-guard 项目

## 连接配置

```
地址：172.17.138.167:8000
认证：master/master
```

## 参见
- [[dashscope]] — 嵌入模型提供方
- [[hybrid-search]] — 混合搜索架构
- [[hnsw-index]] — 向量索引详解
- [[surrealdb-analyzer]] — 自定义分析器

^[raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md]
^[raw/articles/SurrealDB向量集成方案分析.md]
^[raw/articles/SurrealDB索引定义解析.md]
