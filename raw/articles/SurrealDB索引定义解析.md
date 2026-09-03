# SurrealDB 索引定义解析

## 1. 自定义函数：`fn::stripPunct`

```sql
DEFINE FUNCTION OVERWRITE fn::stripPunct($text: string) {
    RETURN string::replace($text, /[\-_!?@#%^&*=+\/\\？！。；''""【】「」￥·×（），、…—]+/, " ");
};
```

**作用**：把中英文标点符号全部替换为空格。

**目的**：在做 ngram 分词前先清理掉干扰字符，避免标点被当作词的一部分影响分词效果。

---

## 2. 自定义分析器：`l3gram`

```sql
DEFINE ANALYZER OVERWRITE l3gram FUNCTION fn::stripPunct TOKENIZERS blank,punct,class FILTERS lowercase,ngram(1,3);
```

**文本处理流水线**（依次执行）：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | `stripPunct` | 去除中英文标点，替换为空格 |
| 2 | `blank` 分词 | 按空格切分 |
| 3 | `punct` 分词 | 按标点切分 |
| 4 | `class` 分词 | 按字符类别（字母/数字/汉字等）切分 |
| 5 | `lowercase` | 全部转小写 |
| 6 | `ngram(1,3)` | 生成 1~3 字符的 ngram（unigram + bigram + trigram） |

**效果举例**：`"hello"` → `h, he, hel, e, el, ell, l, ll, llo, l, lo, o`

**为什么用 ngram(1,3)**：
- 支持模糊/部分匹配，用户搜 `"hel"` 也能匹配 `"hello"`
- 适合中文场景 — 中文没有天然空格分隔，ngram 可以切出有意义的字词组合
- 1~3 字符的粒度兼顾了召回率和精度

---

## 3. `document_chunks` 表索引

### 3.1 全文索引

```sql
DEFINE INDEX OVERWRITE document_chunks_idx ON document_chunks FIELDS content FULLTEXT ANALYZER l3gram BM25 HIGHLIGHTS CONCURRENTLY;
```

| 参数 | 说明 |
|------|------|
| `content` | 索引目标字段 |
| `FULLTEXT` | 全文搜索索引类型 |
| `ANALYZER l3gram` | 使用 l3gram 分析器分词 |
| `BM25` | 评分算法（经典全文检索排序，基于词频和逆文档频率） |
| `HIGHLIGHTS` | 支持返回匹配高亮片段 |
| `CONCURRENTLY` | 异步后台构建，不阻塞写入 |

### 3.2 向量索引

```sql
DEFINE INDEX OVERWRITE document_embedding_idx ON document_chunks FIELDS embedding HNSW DIMENSION 1024 DIST COSINE EFC 100 CONCURRENTLY;
```

| 参数 | 说明 |
|------|------|
| `embedding` | 索引目标字段（向量字段） |
| `HNSW` | 向量近似搜索算法（Hierarchical Navigable Small World） |
| `DIMENSION 1024` | 向量维度为 1024 |
| `DIST COSINE` | 余弦距离度量 |
| `EFC 100` | 查询时扩展因子，越大结果越准但越慢 |
| `CONCURRENTLY` | 异步后台构建 |

---

## 4. `goods_test` 表索引

### 4.1 全文索引

```sql
DEFINE INDEX OVERWRITE goods_text_idx ON goods_test FIELDS goods_name FULLTEXT ANALYZER l3gram BM25 HIGHLIGHTS CONCURRENTLY;
```

对 `goods_name`（商品名称）字段建全文搜索索引。

### 4.2 向量索引

```sql
DEFINE INDEX OVERWRITE goods_vector_idx ON goods_test FIELDS embedding HNSW DIMENSION 1024 DIST COSINE EFC 100 CONCURRENTLY;
```

对 `embedding` 字段建向量搜索索引。

---

## 5. 整体架构：混合检索（Hybrid Search）

| 索引类型 | 用途 | 算法 | 优势 |
|---------|------|------|------|
| **FULLTEXT + BM25** | 关键词精确/模糊匹配 | BM25 词频统计 | 精确匹配、可解释性强、支持高亮 |
| **HNSW + COSINE** | 语义相似度匹配 | 向量余弦距离 | 语义理解、支持同义/近义匹配 |

**查询策略**：同时走全文索引和向量索引，然后将两路结果合并排序，兼顾关键词匹配和语义理解。

---

## 6. 参数调优建议

| 参数 | 当前值 | 说明 |
|------|--------|------|
| `ngram(1,3)` | 1~3 | 可改为 `ngram(2,4)` 减少噪声，但会降低短词召回 |
| `DIMENSION` | 1024 | 需与嵌入模型的输出维度一致（如 text-embedding-v3/v4 支持 1024） |
| `EFC` | 100 | 调大更准（建议 100~200），调小更快（建议 50~80） |
| `DIST` | COSINE | 也可用 `EUCLIDEAN`（欧氏距离），取决于向量归一化方式 |
