---
title: N-gram 分析器 (l3gram)
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [fulltext-search, surrealdb, ngram-analyzer]
sources: [raw/articles/SurrealDB索引定义解析.md]
confidence: high
---

# N-gram 分析器 (l3gram)

在 [[surrealdb]] 中自定义的全文分析器，通过 ngram(1,3) 分词策略支持中文和模糊匹配。

## 定义

```sql
-- 1. 去标点函数
DEFINE FUNCTION OVERWRITE fn::stripPunct($text: string) {
    RETURN string::replace($text, /[\-_!?@#%^&*=+\/\\？！。；''""【】「」￥·×（），、…—]+/, " ");
};

-- 2. 分析器
DEFINE ANALYZER OVERWRITE l3gram FUNCTION fn::stripPunct
    TOKENIZERS blank, punct, class
    FILTERS lowercase, ngram(1,3);
```

## 处理流水线

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | stripPunct | 中英文标点 → 空格 |
| 2 | blank 分词 | 按空格切 |
| 3 | punct 分词 | 按标点切 |
| 4 | class 分词 | 按字符类别切 |
| 5 | lowercase | 转小写 |
| 6 | ngram(1,3) | 生成 1~3 字符的 ngram |

效果：`"hello"` → `h, he, hel, e, el, ell, l, ll, llo, l, lo, o`

## 为什么用 ngram(1,3)

- **模糊匹配**：搜 `"hel"` 能匹配 `"hello"`
- **中文适配**：中文无空格分隔，ngram 切出有意义的字词组合
- **平衡精度**：1~3 字符粒度兼顾召回率和精度

## 调优

- 改为 `ngram(2,4)` 可减少噪声，但降低短词召回
- 去标点用空格替换（非删除）是为了保留词边界

## 使用示例

```sql
DEFINE INDEX idx ON goods FIELDS goods_name FULLTEXT ANALYZER l3gram BM25 HIGHLIGHTS CONCURRENTLY;
```

## 参见
- [[surrealdb]] — 数据库
- [[hybrid-search]] — 混合搜索
- [[bm25]] — 评分算法

^[raw/articles/SurrealDB索引定义解析.md]
