---
title: 混合搜索 (Hybrid Search)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [hybrid-search, surrealdb, vector-search, fulltext-search, architecture]
sources: [raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md, raw/articles/SurrealDB电商混合搜索与多维推荐方案.md, raw/articles/SurrealDB索引定义解析.md]
confidence: high
---

# 混合搜索 (Hybrid Search)

同时使用向量语义搜索和全文关键词搜索，然后融合两路结果的检索架构。在 [[surrealdb]] 中可完全在数据库内完成。

## 架构：硬过滤 + 双路召回 + 业务重排

```
用户输入
    ↓
硬过滤（WHERE stock > 0, goods_status = 1）
    ↓
┌──────────────┬──────────────┐
│ 向量召回      │ 全文召回      │
│ <|N,ef|>     │ @0@ keyword  │
│ 语义理解      │ 精确匹配      │
└──────┬───────┴──────┬───────┘
       ↓              ↓
   search::linear() 或 search::rrf()
       ↓
   业务重排（销量、库存、评分因子）
       ↓
   最终结果
```

## 两路召回

### 向量搜索（语义路）
```sql
LET $vs = SELECT id, (1.0 / (1.0 + vector::distance::knn())) AS score
          FROM product
          WHERE embedding <|20,50|> $query_vector AND stock > 0;
```

### 全文搜索（关键词路）
```sql
LET $ft = SELECT id, search::score(1) AS score
          FROM product
          WHERE title @1@ $keyword AND stock > 0
          ORDER BY score DESC LIMIT 20;
```

## 融合方式

### [[rrf-fusion]] — 倒数排名融合
- 不看分数，只看排名
- 公式：`rrf_score = Σ 1/(k + rank)`，k 默认 60
- 适合分数不可比的场景（余弦相似度 vs BM25）

### [[search-linear]] — 线性加权融合
- Min-Max 归一化后加权求和
- 语法：`search::linear([[$set, 'score_field'], ...], weights, limit, 'minmax')`
- 适合需要精细控制权重的场景

## 电商场景的业务重排

在融合分数基础上，乘法注入业务因子：

| 因子 | 公式 | 作用 |
|------|------|------|
| 销量 | `math::log1p(sales) * 0.1` | 对数平滑，防爆款垄断 |
| 库存 | `IF stock < 3 THEN 0.7 ELSE 1.0` | 低库存降权 |
| 评分 | `IF rating < 4.0 THEN rating/4.0 ELSE 1.0` | 差评降权 |
| 新品 | `IF publish_time > time::now() - 3d THEN 1.2 ELSE 1.0` | 新品扶持 |

### 权重调整指南

| 场景 | 权重 [向量, 文本, 销量] |
|------|------------------------|
| 默认平衡 | [0.6, 0.4, 0.35] |
| 大促冲量 | [0.4, 0.3, 0.5] |
| 精准长尾词 | [0.8, 0.5, 0.1] |

## 参见
- [[two-stage-search-pipeline]] — 两阶段流水线工业级完整实现
- [[surrealdb]] — 数据库实现
- [[hnsw-index]] — 向量索引
- [[rrf-fusion]] — RRF 融合算法
- [[search-linear]] — 线性融合函数
- [[ngram-analyzer]] — 全文分析器

^[raw/articles/Surreal 内置函数（向量搜索和全文搜索）.md]
^[raw/articles/SurrealDB电商混合搜索与多维推荐方案.md]
