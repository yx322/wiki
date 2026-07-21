# SurrealDB DashScope 向量迁移与代码更新指南

本文档记录了将 `knowledge_base` 项目及 SurrealDB 数据库从 Ollama (bge-m3) 迁移至 DashScope (text-embedding-v4) 的完整操作步骤。
**核心策略**：保留旧 `embedding` 字段（备份/对比），新增 `embedding_dashscope` 字段（业务使用）。

---

## 第一部分：SurrealDB 端操作

### 1. 新增向量字段
对需要处理的所有表（如 `document_chunks`, `goods_test` 等）执行：

```sql
-- 为 document_chunks 表新增字段
DEFINE FIELD embedding_dashscope ON document_chunks TYPE array<float>;

-- 为 goods_test 表新增字段（如果有其他表）
DEFINE FIELD embedding_dashscope ON goods_test TYPE array<float>;
```

### 2. 定义底层 API 函数
此函数负责直接调用 DashScope API，返回纯向量数组。

```sql
DEFINE FUNCTION fn::dashscope_embed($text: string) {
    -- 1. 从 config 表读取 API Key
    LET $config = SELECT value FROM config WHERE name = 'dashscope_api_key' LIMIT 1;
    LET $api_key = $config[0].value.api_key;
    
    IF !$api_key {
        THROW "Missing DashScope API Key";
    };

    -- 2. 发起 HTTP POST 请求
    LET $resp = http::post(
        "https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings",
        {
            model: "text-embedding-v4",
            input: [$text]
        },
        {
            Authorization: string::concat("Bearer ", $api_key),
            "Content-Type": "application/json"
        }
    );

    -- 3. 提取并返回向量
    RETURN $resp.data[0].embedding;
};
```

### 3. 定义批量处理函数
针对特定表的循环处理逻辑。

**针对 `document_chunks` 表：**

```sql
DEFINE FUNCTION fn::batch_embed_chunks_dashscope($limit: int ) {
    -- 验证配置
    LET $config = (SELECT value FROM config WHERE name = 'dashscope_api_key' LIMIT 1);
    IF array::len($config) == 0 {
        THROW 'DashScope API Key configuration not found';
    };

    -- 查找 embedding_dashscope 为空的记录
    LET $chunks = (
        SELECT id, content FROM document_chunks
        WHERE content != NONE 
          AND content != '' 
          AND embedding_dashscope = NONE 
        LIMIT $limit
    );

    LET $count = 0;

    FOR $chunk IN $chunks {
        LET $vec = fn::dashscope_embed($chunk.content);
        
        IF $vec {
            UPDATE $chunk.id SET embedding_dashscope = $vec;
            LET $count = $count + 1;
        };
    };

    RETURN { processed: $count, total: array::len($chunks) };
};
```

**针对其他表（如 `goods_test`）：**
你需要复制上面的函数定义，修改两个地方：
1.  函数名改为 `fn::batch_embed_goods_test`。
2.  表名改为 `goods_test`。

### 4. 使用方法
定义好函数后，在 Surrealist 中运行以下命令刷数据：

```sql
-- 每次处理 50 条（循环运行直到 processed 为 0）
RETURN fn::batch_embed_chunks_dashscope(50);
```

---

## 第二部分：Python 项目代码更新 (`knowledge_base`)

### 1. 配置文件更新 (`config.yaml`)

移除或注释旧 `ollama` 配置，新增：

```yaml
dashscope:
  api_key: "sk-你的真实APIKEY"
  embedding_model: "text-embedding-v4"
```

### 2. 设置类更新 (`modules/config/settings.py`)

新增配置类并加入主配置：

```python
class DashScopeSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="DASHSCOPE__",
        env_nested_delimiter="__",
        extra="ignore",
    )
    api_key: str = Field(default="", description="DashScope API Key")
    embedding_model: str = Field(default="text-embedding-v4", description="嵌入模型名称")

# Settings 主类中添加：
dashscope: DashScopeSettings = Field(default_factory=DashScopeSettings)
```

### 3. 检索客户端更新 (`modules/retrieval/retrieval_client.py`)

**关键变更**：
*   SQL 模板改为调用 `fn::dashscope_embed`。
*   搜索字段改为 `embedding_dashscope`。

```python
# RRF_QUERY 修改为：
RRF_QUERY = '''
LET $qvec = fn::dashscope_embed($query_param);
LET $vs = SELECT id, file_name, content FROM {table}
          WHERE embedding_dashscope <|{limit},COSINE|> $qvec;
...
'''

def __init__(self):
    self.embedding_model: str = settings.dashscope.embedding_model
```

### 4. 向量化流水线更新 (`modules/pipelines/vectorize.py`)

**关键变更**：
*   请求目标改为 DashScope URL。
*   载荷改为 `input: [text]`，解析路径为 `data[0].embedding`。
*   保存的 JSON key 改为 `embedding_dashscope`。

### 5. 存储与索引更新 (`modules/pipelines/store.py`)

**关键变更**：
*   建表语句增加 `embedding_dashscope` 字段定义。
*   索引创建语句改为：

```python
index_sql = f"DEFINE INDEX vector_idx_dashscope ON {vector_table} FIELDS embedding_dashscope HNSW DIMENSION 1024 DIST COSINE EFC 100"
```

### 6. 文档处理 API 更新 (`modules/api/documents/document.py` & `webhook.py`)

**`document.py`**: `vectorize_text` 函数改为调用 DashScope API。
**`webhook.py`**: 新增 `_vectorize_batch_dashscope` 方法，移除对旧 Ollama 客户端的依赖，确保新生成的向量数据 key 为 `embedding_dashscope`。
