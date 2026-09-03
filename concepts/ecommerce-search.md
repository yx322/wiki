---
title: 电商搜索与推荐架构
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [hybrid-search, ecommerce, recommendation, surrealdb]
sources: [raw/articles/ecommerce-recommendation-schemes.md, raw/articles/ecommerce-search-architecture.md]
confidence: high
---

# 电商搜索与推荐架构

从"纯相关性权重"到"相关性门槛原则"，再到"准乘法融合"的架构演进。

## 向量与文本权重 (0.6 vs 0.4)

| 维度 | 权重 | 核心职责 | 典型场景 |
|------|------|---------|---------|
| 向量搜索 | 0.6 | 懂意图、找同义、抗拼写错误 | 搜"透气跑鞋"→召回"网面运动鞋" |
| 全文搜索 | 0.4 | 锚定精确词、型号、SKU、品牌 | 搜"iPhone 15 Pro"→精确命中型号 |

## 相关性门槛原则

**核心诉求**：彻底贯彻"Relevance First, Business Second"，防止业务权重逆袭导致不相关商品排在前面。

### 方案一：权重绝对压制
强制让相关性权重占比 > 70%，业务权重只能做微调。

### 方案二：乘法融合（工业级标准）
`最终得分 = 相关性得分 × (1 + 业务增益)`

**数学保证**：相关性为 0，最终得分必为 0。业务因子只能"放大"相关商品，绝不可能把不相关商品"无中生有"顶到前排。

### 推荐：基于阈值的"准乘法"

```sql
LET $final_score = $base_score + (
    IF $base_score > 0.5 THEN
        (sales_score * 0.2 + stock_score * 0.1 + push_score * 0.25)
    ELSE
        0.0
    END
);
```

## 5 步搜索全流程

1. **双路召回**：向量路（语义相似）+ 全文路（字面匹配），"宁可错杀，不可漏过"
2. **计算纯相关性**：仅 vs_score 和 ft_score 加权（0.6 vs 0.4），归一化到 [0,1]
3. **相关性门槛**：基础相关分 > 0.5 才允许业务加分，否则业务分清零
4. **业务增益**：销量（log1p 防爆款垄断）+ 库存 + 主推
5. **最终排序**：按最终得分降序，截取前 N

## 加权 RRF vs 两阶段排序

| 方案 | 核心原理 | 优点 | 缺点 |
|------|---------|------|------|
| **加权 RRF** | 业务因素作为独立检索器，RRF 融合加权 | 结构清晰、归一化自然 | 排名平滑抹平极端差异 |
| **两阶段排序** | RRF 召回候选集，应用层复杂公式重排 | 灵活性强、计算高效 | 可能漏掉好商品 |

**库存处理**：用 `WHERE stock_qty > 0` 作为过滤条件，不参与计分。

## 参见
- [[two-stage-search-pipeline]] — 两阶段流水线工业级完整实现
- [[hybrid-search]] — 混合搜索架构
- [[surrealdb]] — 数据库实现
- [[rrf-fusion]] — RRF 融合算法
- [[search-linear]] — 线性融合

^[raw/articles/ecommerce-recommendation-schemes.md]
^[raw/articles/ecommerce-search-architecture.md]
