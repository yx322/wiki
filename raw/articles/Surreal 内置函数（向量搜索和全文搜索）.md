# DeepSeek 分享对话

> 来源: https://chat.deepseek.com/share/hzldr8p71fdjwmew13


---


## 👤 用户


在 SurrealDB 中，利用其原生的 混合搜索函数（如 search::linear 或 search::rrf）、内置数学函数以及强大的条件过滤能力。给我详细解释一下


## 🤖 DeepSeek


在 SurrealDB 中，要构建一个强大的混合搜索系统，核心是巧妙地结合其向量搜索和全文搜索功能，并通过融合函数将两者结果合并[9]。下面我为你详细拆解这三个核心部分。

### 🔍 实现混合搜索的三个核心组件

整个实现过程可以概括为三个步骤：定义数据模型、执行并行搜索、融合结果排名。

#### 1. 📊 定义数据模型

首先，你需要为数据表设计好字段，以支持两种搜索方式。

-   **全文搜索**：定义一个 `FULLTEXT` 类型的索引，它会对指定字段的内容进行分词，为后续的BM25关键词搜索做准备[3][4]。
    ```sql
    -- 定义一个名为 'article_ft' 的全文索引，作用于 'content' 字段
    DEFINE INDEX article_ft ON TABLE article FIELDS content FULLTEXT;
    ```

-   **向量搜索**：定义一个类型为 `HNSW` 的向量索引。你需要指定向量维度（比如使用OpenAI的1536维模型）和一个距离函数（如`COSINE`）。HNSW索引是实现高效近似最近邻（ANN）搜索的关键，它能将向量搜索的复杂度从O(N)降至对数级别[3][4][9]。
    ```sql
    -- 定义一个名为 'article_vs' 的 HNSW 向量索引，作用于 'embedding' 字段
    DEFINE INDEX article_vs ON TABLE article FIELDS embedding MTREE DIMENSION 1536;
    ```

#### 2. ⚡️ 执行并行搜索

混合搜索的威力在于让两种搜索方式同时工作，然后合并结果。在SurrealQL中，你可以用两个`LET`语句来分别执行查询[4][6]。

-   **向量语义搜索 (`$vs`)**：使用`<||>`操作符在HNSW索引上进行快速ANN搜索。这个查询旨在理解用户查询的“意图”和“语义”[3][4]。
    ```sql
    -- 假设 $query_embedding 是你的查询向量
    LET $vs = SELECT id, 
                     vector::similarity::cosine(embedding, $query_embedding) AS score
              FROM article
              WHERE embedding <|3,100|> $query_embedding; -- 返回3个结果，搜索候选池为100
    ```

-   **关键词精确搜索 (`$ft`)**：使用`@@`操作符和`search::score()`函数进行BM25关键词搜索。这个查询侧重于匹配用户输入的具体词汇[1][4]。
    ```sql
    -- 假设 $keyword 是用户输入的查询文本
    LET $ft = SELECT id, 
                     search::score(0) + search::score(1) AS score -- 合并多个字段的分数
              FROM article
              WHERE content @0@ $keyword -- @0@ 对应 search::score(0)
                 OR title @1@ $keyword  -- @1@ 对应 search::score(1)
              ORDER BY score DESC
              LIMIT 10;
    ```

#### 3. 🧬 融合结果排名 (`search::rrf()`)

这是混合搜索的“大脑”，也是 SurrealDB 的精妙之处。`search::rrf()` 函数采用**倒数排名融合（Reciprocal Rank Fusion, RRF）**算法，解决了直接合并分数（如余弦相似度0.8和BM25分数12.5）无法比较的难题[1][4]。

-   **核心逻辑**：该算法不看分数，只看排名。它会将`$vs`和`$ft`返回的两个结果集合并，然后对每个结果项计算其`rrf_score`，这个分数由该项在不同结果集中的排名决定[4][6]。

-   **计算公式**：`rrf_score = Σ 1 / (k + rank)`
    -   `k` 是平滑常数，默认为`60`，用于防止排名靠后的项贡献过小[1][4]。
    -   这意味着，如果一个结果在两种搜索中排名都较高，其最终得分会显著更高。

-   **最终查询**：执行融合操作，按`rrf_score`排序并返回Top K个最相关的文档[1][8]。
    ```sql
    -- 融合前面两个查询的结果，返回最终排名前5的结果
    RETURN search::rrf([$vs, $ft], 5, 60);
    ```

### 💡 完整实战代码示例

下面是一个完整的示例，它将向量搜索、关键词搜索和`search::rrf()`融合在一个查询中，你可以将其作为模板。

```sql
-- 1. 定义并设置查询向量和关键词
-- 在实际应用中，你需要通过外部服务（如OpenAI）将用户问题转换为embedding向量
LET $query_embedding = [0.1, 0.2, ...]; -- 代表用户问题的向量
LET $keyword = "你的搜索关键词";

-- 2. 执行并行搜索
-- 2.1 向量搜索（语义理解）
LET $vs = SELECT id, 
                 vector::similarity::cosine(embedding, $query_embedding) AS score
          FROM article
          WHERE embedding <|5,100|> $query_embedding;

-- 2.2 全文搜索（关键词匹配）
LET $ft = SELECT id, 
                 search::score(0) AS score
          FROM article
          WHERE content @0@ $keyword
          ORDER BY score DESC
          LIMIT 10;

-- 3. 融合结果（RRF）
RETURN search::rrf([$vs, $ft], 10, 60);
```

### 📈 拓展应用：融合后的结果再增强

`search::rrf()`返回的只是文档ID和融合分数。当你获得这份高度相关的文档列表后，可以利用SurrealDB的原生图数据库能力进行**结果的上下文增强**[4][6]。

例如，在代码搜索场景中，`search::rrf()`找到了一个相关的`function`。你可以紧接着查询它的**父类**、**同级的其他函数**、以及它的**调用链**。所有这些都可以在一次数据库交互中完成，无需多次请求。

```sql
-- 在获得 $rrf_results （即 search::rrf 的结果）后
-- 对每个结果进行图遍历，获取更多上下文
SELECT
    id,
    (SELECT name FROM class WHERE file = function.file ORDER BY lineno DESC LIMIT 1).name AS class,
    (SELECT name FROM function WHERE class = class LIMIT 5) AS sibling_functions,
    (SELECT <-calls<-function.name AS callers, ->calls->function.name AS callees)
FROM $rrf_results;
```

SurrealDB 的混合搜索让开发者能用一个查询在数据库内部完成语义与关键词的融合，兼顾了易用性和性能。这种模式避免了复杂的应用层逻辑，让向量与全文检索的互补优势得以充分发挥。
