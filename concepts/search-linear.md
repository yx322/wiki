---
title: search::linear (线性加权融合)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [search-linear, hybrid-search, surrealdb, performance]
sources: [raw/articles/SurrealDB电商混合搜索与多维推荐方案.md]
confidence: high
---

# search::linear (线性加权融合)

[[surrealdb]] 的多路搜索结果线性融合函数，支持 Min-Max 归一化和自定义权重。比 [[rrf-fusion]] 更精细。

## 语法

```sql
search::linear(
    [
        [$result_set, 'score_field_name'],
        [$result_set2, 'score_field_name2'],
        ...
    ],
    [weight1, weight2, ...],
    limit,
    'minmax'  -- 归一化方式
)
```

## 关键：必须绑定分数字段

❌ 错误写法（直接传结果集，会报错）：
```sql
search::linear([$vs, $ft], [0.6, 0.4], 20, 'minmax')
```

✅ 正确写法（指定分数字段名）：
```sql
search::linear(
    [[$combined, 'vs_score'], [$combined, 'ft_score']],
    [0.6, 0.4],
    20,
    'minmax'
)
```

## Min-Max 归一化

`'minmax'` 参数会自动计算每个列表中的最大/最小值，将所有分数等比例缩放到 [0, 1] 区间后再加权求和。

**为什么必须归一化**：BM25 分数可能是 12.5，余弦相似度是 0.8，库存是 1000。不归一化的话，库存数值会彻底碾压其他分数。

## 电商场景实例

三路融合（向量 + 文本 + 销量）：

```sql
LET $ranked = search::linear(
    [
        [$combined, 'vs_score'],
        [$combined, 'ft_score'],
        [$combined, 'sales_score']  -- math::log1p(sales)
    ],
    [0.6, 0.4, 0.35],
    20,
    'minmax'
);
```

权重占比：向量 38.7% + 文本 25.8% + 销量 22.5% = 相关性总计 87%。

## 参见
- [[hybrid-search]] — 混合搜索架构
- [[rrf-fusion]] — 排名融合
- [[surrealdb]] — 数据库

^[raw/articles/SurrealDB电商混合搜索与多维推荐方案.md]
