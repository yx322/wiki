# SurrealDB 向量处理与 DashScope 集成方案分析

本文档基于最新的代码审查，深度解析了 SurrealDB 中向量处理的流水线架构，并提供了**纯 DashScope（去除 Ollama 依赖）**的完整实现方案。

---

## 第一部分：核心代码模块深度解析

以下是对原始代码中五个关键逻辑段的拆解分析，涵盖了从配置管理、API 集成到数据清洗的全流程。

### 1. 配置管理：`UPSERT` 与系统解耦

```sql
UPSERT config SET
    name = 'ollama',
    type = 'embedding',
    `value` = {
        base_url: $base_url ?? 'http://localhost:11434',
        timeout: $timeout ?? 30000,
        options: $options ?? {},
    }
WHERE name = 'ollama' AND type = 'embedding';
```

*   **设计意图**：实现“控制与执行分离”。将外部服务的连接信息（IP、端口、超时）存储在数据库中，而非硬编码在代码里。
*   **核心机制**：
    *   **UPSERT (Update + Insert)**：保证操作的**幂等性**。无论执行多少次，系统中只存在一份有效的 Ollama 配置，避免产生重复数据。
    *   **防御性编程 (`??`)**：空值合并运算符确保即使调用时未传参，系统也能使用安全的默认值（如 `localhost:11434`），提高系统健壮性。

### 2. API 集成层：DashScope HTTP 请求

```sql
LET $resp = http::post(
    'https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings',
    {
        model: 'text-embedding-v4',
        input: [$text],
    },
    { Authorization: 'Bearer sk-...', "Content-Type": 'application/json' }
);
RETURN $resp.data[0].embedding;
```

*   **协议对齐**：`input: [$text]` 是关键。DashScope (兼容 OpenAI 标准) 要求输入必须是**列表**。即使只有一句话，也必须用中括号包裹。
*   **精准提取**：
    *   `$resp.data`：获取响应体中的数组。
    *   `[0]`：定位到第一条结果（因为我们只发了一条请求）。
    *   `.embedding`：剥离外层 JSON 包装，直接提取 1024 维的浮点数组。

### 3. 业务层：`FOR` 循环批量处理（刷向量）

```sql
FOR $chunk IN $chunks {
    LET $response = fn::ollama::embed($model, $chunk.content);
    IF $response AND $response.embeddings {
        UPDATE $chunk.id SET embedding = $response.embeddings[0];
        LET $count = $count + 1;
    };
};
```

*   **工作流**：查询未处理文本 $\rightarrow$ 逐条调用 Ollama $\rightarrow$ 向量回写数据库。
*   **架构隐患（长事务警告）**：
    *   此逻辑通常在单一事务中运行。若处理 1000 条数据，耗时可能长达 8 分钟。
    *   **风险**：一旦中间某条失败（如网络抖动），整个事务回滚，导致前功尽弃。
    *   **建议**：生产环境建议使用 `LIMIT` 分批执行，或迁移至 Python 脚本异步处理。

### 4. 动态适配层：手动 Ollama 调用

*   **原理**：展示如何在不依赖 SurrealDB 内置 `fn::ollama` 插件的情况下，通过读取 `config` 表中的 `base_url`，利用 `string::concat` 拼接地址并发起 `http::post`。
*   **价值**：提供了极大的灵活性，允许连接任意 IP 上的 Ollama 实例。

### 5. 数据清洗层：Regex 正则过滤

```sql
RETURN string::replace($text, /[\-_!?@#%^&*=+\/\\？！。；...]+/, ' ');
```

*   **逻辑**：匹配所有中英文标点及特殊符号，替换为**空格**。
*   **为何用空格替换？**
    *   **保留词边界**：若 `安全!帽` 被替换为 `安全帽`（粘连），可能被 Tokenizer 视为一个新词。
    *   **正确做法**：替换为 `安全 帽`（空格分隔），模型能更准确地理解语义，减少噪声干扰。

---

## 第二部分：纯 DashScope 方案（去 Ollama 版）

如果决定**完全弃用 Ollama**，仅依赖阿里云 DashScope，系统将更轻量，无需维护本地模型，仅需管理 API Key。以下是针对该方案的四个详细步骤：

### 第一步：定义核心函数（代替 Ollama 内置函数）

由于 SurrealDB 没有内置 `fn::dashscope::embed`，我们需要先定义一个函数，把 `http::post` 封装起来。

**代码逻辑**：
```sql
DEFINE FUNCTION fn::dashscope_embed($text: string) {
    -- 1. 从数据库读取 API Key（安全做法，避免硬编码）
    LET $config = (SELECT value FROM config WHERE name = 'dashscope_api_key' LIMIT 1);
    LET $api_key = $config[0].value;
    
    IF !$api_key {
        THROW "Missing DashScope API Key. Please configure it first.";
    }

    -- 2. 发起 HTTP 请求
    LET $resp = http::post(
        "https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings",
        {
            model: "text-embedding-v4", -- DashScope 模型
            input: [$text]              -- 注意：必须是列表格式
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

**详细解析**：
1.  **安全配置**：通过 `SELECT` 从 `config` 表动态获取 Key。如果配置不存在，通过 `THROW` 抛出错误，防止请求因缺少 Key 而失败。
2.  **输入格式**：`input: [$text]` 使用数组包裹文本，这是 DashScope API 的标准要求（支持批量输入）。
3.  **返回值**：直接解析 JSON 响应，取出 `data` 数组中第一项的 `embedding` 字段，返回纯净的向量数组 `[0.012, ...]`。

### 第二步：保存 DashScope API Key

```sql
-- 推荐：直接创建指定 ID 的记录
CREATE config:dashscope_api_key SET
    name = 'dashscope_api_key',
    value = 'sk-你的真实APIKEY';  -- 替换为真实 Key

RETURN "DashScope API Key configured.";
```

**说明**：
- 不需要 `base_url` 和 `timeout`，DashScope 端点固定
- 用 `CREATE config:dashscope_api_key` 而非 `UPSERT`，因为该记录尚不存在

### 第三步：修改“批量刷向量”脚本

这是改动最大的地方，去除了对本地模型的依赖和复杂的 URL 拼接。

**代码逻辑**：
```sql
-- 1. 参数准备
LET $limit_val = $limit ?? 100;

-- 2. 查询需要处理的文本
LET $chunks = (
    SELECT id, content
    FROM document_chunks
    WHERE content != NONE AND content != ''
    -- 可选优化：AND embedding = NONE 避免重复计算已处理的记录
    LIMIT $limit_val
);

LET $count = 0;

-- 3. 核心循环：调用 DashScope
FOR $chunk IN $chunks {
    -- 调用上面定义的函数，直接获取向量
    LET $vec = fn::dashscope_embed($chunk.content);
    
    -- 检查是否成功拿到向量（数组不为空）
    IF $vec {
        UPDATE $chunk.id SET embedding = $vec;
        LET $count = $count + 1;
    };
};

-- 4. 返回结果统计
RETURN { processed: $count, total: array::len($chunks) };
```

**详细解析**：
1.  **逻辑简化**：移除了原有的 `LET $ollama_config = ...` 和 `fn::ollama::embed` 调用。
2.  **直接调用**：`fn::dashscope_embed` 内部已经处理了 HTTP 请求和 JSON 解析，这里只需传入文本即可拿到向量。
3.  **数据保护**：循环中增加了对 `$vec` 的存在性检查，确保只有成功获取向量时才更新数据库。

### 第四步：搜索查询（更优雅）

不再需要拼接 URL 或处理 Ollama 的配置对象，搜索逻辑变得非常直观。

**代码逻辑**：
```sql
-- 1. 获取查询词向量
LET $qvec = fn::dashscope_embed("推荐安全帽");

-- 2. 执行向量搜索
LET $vs = SELECT id, content
          FROM document_chunks
          WHERE embedding <|10,COSINE|> $qvec;

RETURN $vs;
```

**详细解析**：
*   **零配置调用**：用户不需要知道任何模型名称或 API 地址，只需传入搜索词 `"推荐安全帽"`，底层函数会自动完成所有网络交互。
*   **无缝集成**：返回的 `$qvec` 直接是一个 1024 维的浮点数数组，可以被 `<|10,COSINE|>` 算子直接识别用于相似度匹配。

---

## 第三部分：双模并行方案（同时使用 Ollama + DashScope）

如果你的业务场景**既想用 Ollama 的本地速度**，又**想用 DashScope 的高质量**（例如：日常搜索用本地，疑难搜索用云端；或者为了做 A/B 测试对比效果），可以采用“双模并行”方案。

这种方案的核心思想是：**数据预处理时同时生成两套向量，搜索时按需选择。**

### 架构设计
1.  **双向量存储**：在 `document_chunks` 表中存储两个字段：
    *   `vec_ollama`：Ollama 生成的向量
    *   `vec_dashscope`：DashScope 生成的向量
2.  **动态路由**：在查询时，通过读取配置决定使用哪个向量进行搜索。

---

### 第一步：配置管理（存储双份配置）

我们需要在 `config` 表中分别保存两套配置，互不冲突。

```sql
-- 1. 保存 Ollama 配置
UPSERT config SET
    name = 'ollama',
    type = 'embedding',
    value = {
        base_url: 'http://localhost:11434',
        model: 'bge-m3'
    }
WHERE name = 'ollama';

-- 2. 保存 DashScope 配置
UPSERT config SET
    name = 'dashscope',
    type = 'embedding',
    value = {
        api_key: 'sk-你的真实 Key',
        model: 'text-embedding-v4'
    }
WHERE name = 'dashscope';

RETURN "Dual configuration set.";
```

---

### 第二步：批量处理（生成并存储两套向量）

在刷数据脚本中，针对同一条文本，分别调用两次不同的函数，并将结果存入不同的字段。

```sql
LET $limit_val = $limit ?? 100;

LET $chunks = (
    SELECT id, content
    FROM document_chunks
    WHERE content != NONE
    -- 过滤掉已经处理过的（可选）
    AND vec_ollama = NONE 
    AND vec_dashscope = NONE
    LIMIT $limit_val
);

LET $count = 0;

FOR $chunk IN $chunks {
    -- 1. 调用 Ollama (内置函数)
    -- 注意：如果 Ollama 挂了，可以用 try/catch 处理或设为 null
    LET $vec_o = (fn::ollama::embed('bge-m3', $chunk.content)).embeddings[0];

    -- 2. 调用 DashScope (自定义函数)
    LET $vec_d = fn::dashscope_embed($chunk.content);

    -- 3. 同时写入两个字段
    IF $vec_o AND $vec_d {
        UPDATE $chunk.id SET 
            vec_ollama = $vec_o,
            vec_dashscope = $vec_d;
        LET $count = $count + 1;
    };
};

RETURN { processed: $count, total: array::len($chunks) };
```

*   **优势**：数据只存一份，但拥有两种“视图”。你可以随时对比这两种向量检索出的结果哪个更准。

---

### 第三步：统一搜索（策略模式）

通过一段动态脚本，根据当前的策略配置（`strategy`）决定是查本地还是查云端。

```sql
-- 1. 获取当前激活的策略 (例如：'ollama', 'dashscope', 'both')
LET $cfg = (SELECT value FROM config WHERE name = 'search_strategy' LIMIT 1);
LET $strategy = $cfg[0].value ?? 'ollama';

-- 2. 获取查询词的向量（根据策略）
LET $qvec = IF $strategy == 'dashscope' {
    fn::dashscope_embed($query_text)
} ELSE {
    (fn::ollama::embed('bge-m3', $query_text)).embeddings[0]
};

-- 3. 执行搜索
LET $results = SELECT id, content 
FROM document_chunks 
WHERE 
    -- 根据策略选择搜索的字段
    IF $strategy == 'dashscope' {
        vec_dashscope <|10,COSINE|> $qvec
    } ELSE {
        vec_ollama <|10,COSINE|> $qvec
    };

RETURN $results;
```

### 总结：为什么需要双模方案？

| 场景 | 行为 | 优势 |
| :--- | :--- | :--- |
| **日常低延迟** | 策略设为 `ollama` | 走本地网络，速度极快，不消耗 API 额度 |
| **高质量召回** | 策略切换为 `dashscope` | 利用云端大模型的语义理解能力，解决本地搜不准的词 |
| **效果验证** | 存储双份数据 | 可以在后台跑 A/B Test，分析哪个模型更适合你的业务数据 |
| **容灾备份** | 自动降级 | 代码逻辑可改为：如果 Ollama 搜索结果为空，自动用 DashScope 再搜一次 |

---

## 第四部分：方案对比总结

| 维度 | Ollama (本地模型) | DashScope (云端 API) | 双模并行方案 |
| :--- | :--- | :--- | :--- |
| **基础设施** | 需部署 Ollama (占用显存/内存) | **零部署**，仅依赖网络 | 两者兼顾，互为备份 |
| **配置复杂度** | 需配置 IP、端口、模型名 | 仅需配置 **API Key** | 稍复杂，需维护两套配置 |
| **集成方式** | `fn::ollama::embed` (内置) | `fn::dashscope_embed` (自定义) | 动态路由，灵活切换 |
| **性能瓶颈** | 硬件算力 (GPU/CPU 满载) | 网络延迟 (RTT) + API QPS 限制 | **互补**：本地抗并发，云端提质量 |
| **适用场景** | 数据敏感、内网环境、高并发 | 快速启动、追求语义质量、不想管服务器 | A/B 测试、容灾降级、追求极致效果 |

---

## 第五部分：实战迁移指南（`knowledge_base` 项目改造）

这是将 **`D:\knowledge_base`** 项目从 Ollama 切换到纯 DashScope 的完整代码修改指南。

### 第零步：历史数据清洗与重新向量化 (Data Cleaning)

**⚠️ 严重警告**：
正如你所指出的，由于 Ollama (bge-m3) 和 DashScope (text-embedding-v4) 的模型架构不同，**它们生成的向量处于完全不同的坐标空间**。
*   **后果**：修改代码后，**原有的 `document_chunks` 表中的 `embedding` 数据（由 Ollama 生成）将立即失效**。
*   **现象**：如果不重新生成向量，搜索功能的余弦相似度计算将毫无意义（接近 0），导致**搜不到任何结果**。
*   **必须行动**：必须在正式运行搜索前，清洗或重新向量化数据。

**清洗方案（二选一）**：

1.  **方案 A：清空重跑（推荐，有源文件）**
    *   清空表数据：`DELETE FROM document_chunks;`
    *   重新运行 Python 导入脚本：`python run_pipeline.py`。
    *   *优势*：代码已改好，自动用 DashScope 生成正确的向量。

2.  **方案 B：原地刷新（无源文件）**
    *   如果只剩数据库数据，需在 Surrealist 循环调用新函数覆盖旧向量。

**原地刷新脚本（Surrealist）**：
```sql
-- 1. 清除旧向量
UPDATE document_chunks UNSET embedding;

-- 2. 分批调用 DashScope 重新计算（单批手动版，需反复执行）
LET $batch_size = 50;
LET $chunks = SELECT id, content FROM document_chunks WHERE embedding = NONE LIMIT $batch_size;

FOR $chunk IN $chunks {
    LET $vec = fn::dashscope_embed($chunk.content);
    IF $vec {
        UPDATE $chunk.id SET embedding = $vec;
    };
};
```

**⚡ 改进版：全自动 WHILE 循环 + 错误处理 + 进度跟踪**（推荐）

单批版本需要手动反复执行，改进版用 `WHILE` 循环自动处理完所有数据，并加入错误捕获和 API 限流保护：

```sql
-- 持续处理直到全部完成
LET $batch_size = 50;
LET $total_processed = 0;

WHILE (SELECT count() FROM document_chunks WHERE embedding = NONE) > 0 {
    LET $chunks = SELECT id, content 
                  FROM document_chunks 
                  WHERE embedding = NONE 
                  LIMIT $batch_size;
    
    FOR $chunk IN $chunks {
        TRY {
            LET $vec = fn::dashscope_embed($chunk.content);
            UPDATE $chunk.id SET embedding = $vec;
            LET $total_processed = $total_processed + 1;
        } CATCH {
            -- 记录失败的 ID，稍后重试
            -- 也可以写入日志表：INSERT INTO failed_logs { id: $chunk.id, time: time::now() };
        };
    };
    
    -- 输出进度
    LET $remaining = (SELECT count() FROM document_chunks WHERE embedding = NONE);
    PRINT "已处理: " + $total_processed + "，剩余: " + $remaining;
    
    SLEEP 0.1; -- 避免 API 限流
};
```

**改进点**：
- `WHILE` 循环自动处理全表，无需手动重复执行
- `TRY/CATCH` 捕获 API 调用失败，不会中断整个流程
- 进度输出：实时显示已处理和剩余数量
- `SLEEP 0.1`：避免短时间密集请求触发 DashScope 限流

---

### 迁移前备份策略

在正式替换向量前，建议先备份旧数据，以便回滚：

```sql
-- 备份全表到临时表
CREATE document_chunks_backup AS SELECT * FROM document_chunks;
```

---

### 最终确定方案：双向量并行

**方案**：保留原 `embedding` 字段（Ollama bge-m3），新增 `embedding_dashscope` 字段（DashScope `text-embedding-v4`）。

```sql
-- ============================
-- 第一步：新增 DashScope 向量字段
-- ============================
DEFINE FIELD embedding_dashscope ON document_chunks TYPE array<float>;

-- ============================
-- 第二步：利用 content 重新计算 DashScope 向量
-- ============================
LET $batch_size = 50;
LET $total_processed = 0;

WHILE (SELECT count() FROM document_chunks WHERE embedding_dashscope = NONE) > 0 {
    LET $chunks = SELECT id, content 
                  FROM document_chunks 
                  WHERE embedding_dashscope = NONE 
                  LIMIT $batch_size;
    
    FOR $chunk IN $chunks {
        TRY {
            LET $vec = fn::dashscope_embed($chunk.content);
            UPDATE $chunk.id SET embedding_dashscope = $vec;
            LET $total_processed = $total_processed + 1;
        } CATCH {
            -- API 失败跳过，后续可重试
        };
    };
    
    LET $remaining = (SELECT count() FROM document_chunks WHERE embedding_dashscope = NONE);
    PRINT "已处理: " + $total_processed + "，剩余: " + $remaining;
    
    SLEEP 0.1; -- 避免 API 限流
};
```

**执行后每条记录的数据结构**：
```
{
    chunk_id: '乳胶发泡_..._chunk_2',
    content: '| 乳胶过敏、异味刺鼻 | 1. 标注乳胶成分警示...',    -- 不变
    embedding: [-0.8005, 0.0285, -1.1236, ...],              -- Ollama bge-m3（旧）
    embedding_dashscope: [0.1234, -0.8765, 0.4567, ...]      -- DashScope v4（新）
}
```

**迁移验证完成后的搜索代码**（只用 DashScope，不再使用 Ollama 搜索）：
```sql
-- 查询词向量化（DashScope v4）
LET $qvec = fn::dashscope_embed($query);

-- 向量搜索（使用 embedding_dashscope 字段）
SELECT id, content FROM document_chunks 
WHERE embedding_dashscope <|10,COSINE|> $qvec;
```

> **说明**：迁移验证完成后，项目搜索逻辑全部走 `embedding_dashscope` + DashScope v4。Ollama 搜索代码可逐步移除。`embedding`（旧字段）仅保留作为数据备份，不再参与业务查询。

确认新模型稳定后，可清理旧字段释放空间：
```sql
-- 确认没问题后删除旧字段
REMOVE FIELD embedding ON document_chunks;
-- 或重命名保留
UPDATE document_chunks SET embedding_v1_legacy = embedding UNSET embedding;
```

---

### 第一步：修改配置文件 `config.yaml`

将 Ollama 的本地配置替换为 DashScope 的云端配置。

**修改位置：** 根目录下的 `config.yaml`。

```yaml
# 修改前
ollama:
  base_url: "http://localhost:11434"
  embedding_model: "bge-m3"

# 修改为
dashscope:
  api_key: "sk-你的真实 API_KEY"  # <--- 填入 Key
  embedding_model: "text-embedding-v4"
  base_url: "https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings"
```

### 第二步：修改 Python 配置加载 `settings.py`

**修改位置：** `modules/config/settings.py`。
将 `OllamaSettings` 类改为（或新增） `DashScopeSettings`，以支持读取 `api_key`。

```python
class DashScopeSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="DASHSCOPE__",
        env_nested_delimiter="__",
        extra="ignore",
    )
    api_key: str = Field(default="", description="DashScope API Key")
    embedding_model: str = Field(default="text-embedding-v4")
```

同时在主 `Settings` 类中将 `ollama: OllamaSettings` 替换为 `dashscope: DashScopeSettings`。

### 第三步：修改搜索逻辑 `retrieval_client.py`

**修改位置：** `modules/retrieval/retrieval_client.py` 第 39 行 `RRF_QUERY`。
将 SQL 模板中的 Ollama 函数调用替换为 DashScope 函数。

```python
# 修改前
RRF_QUERY = '''
LET $qvec = fn::ollama::embed($model_param, $query_param).embeddings[0];
...
'''

# 修改后
RRF_QUERY = '''
-- 直接调用自定义函数，无需指定模型名，无需剥壳
LET $qvec = fn::dashscope_embed($query_param);
...
'''
```

**核心区别**：
1.  **去掉了 `.embeddings[0]`**：因为 `fn::dashscope_embed` 内部已经提取并返回了纯数组。
2.  **去掉了 `$model_param`**：模型名已在函数定义中指定。

### 第四步：修改向量化流水线 `vectorize.py`

**修改位置：** `modules/pipelines/vectorize.py`。
修改 `vectorize_text_with_ollama_async` 函数，将 HTTP 请求目标从 Ollama 改为 DashScope。

```python
# 1. 修改 Headers 和 Payload
headers = {
    "Authorization": f"Bearer {settings.dashscope.api_key}",
    "Content-Type": "application/json"
}
payload = {
    "model": "text-embedding-v4",
    "input": [text]  # DashScope 要求 input 必须是列表
}

# 2. 发送请求
async with session.post(settings.dashscope.base_url, json=payload, headers=headers, ...):
    result = await response.json()
    # 关键：DashScope 的向量在 data[0].embedding 路径下，不同于 Ollama
    embedding = np.array(result["data"][0]["embedding"], dtype=np.float32)
```

---

## 第六部分：去除 Ollama 后的架构变更总结（Checklist）

本节总结了将系统完全切换为纯 DashScope 方案后的核心变更、风险点及实施清单。

### 1. 核心架构变更
*   **计算位置转移**：向量计算从 **本地服务器** (WSL/Ollama) 转移到了 **阿里云云端** (DashScope)。
*   **依赖解除**：不再需要维护本地 Ollama 进程、Docker 容器或模型权重文件（如 `bge-m3`），系统更轻量。
*   **网络依赖增强**：系统现在强依赖外网连接。如果网络中断，搜索和入库功能将不可用（Ollama 方案可离线运行）。

### 2. ⚠️ 关键风险：向量空间隔离
*   **不兼容性**：正如之前强调的，Ollama 生成的旧向量与 DashScope 生成的新向量在数学上**完全不兼容**（相似度 ≈ 0）。
*   **必须动作**：切换方案后，**绝对不能**直接用新搜索逻辑去查旧向量。**必须**先清洗数据（删除旧向量或重刷全表）。

### 3. 性能与成本影响
| 维度 | Ollama (旧方案) | DashScope (新方案) | 影响评估 |
| :--- | :--- | :--- | :--- |
| **延迟** | **50-150ms** (本地推理) | **50-150ms** (云端 API) | **速度相当**。云端的 TLS 开销与本地的计算开销基本抵消。<br>*(注：之前测试日志中的 500ms+ 是因代码频繁重建 HTTP 连接导致的，非真实性能)* |
| **并发能力** | 低 (受限于本地显存/核数，高并发易排队) | 极高 (云端自动扩容，抗高并发) | DashScope 在高负载下表现远好于本地 Ollama |
| **运维成本** | 高 (需维护进程、模型权重、显卡驱动) | 低 (零运维，纯 API 调用) | **显著降低运维负担** |
| **资金成本** | 硬件投入 (电费/服务器) | 按量付费 (Token 计费) | 初期零成本，长期按使用量付费 |

### 4. 实施 Checklist（实施前请核对）

- [x] **SurrealDB 函数定义**：已执行 `DEFINE FUNCTION fn::dashscope_embed(...)`，确认模型为 `text-embedding-v4`。
- [x] **Key 安全**：已获取 DashScope API Key 并正确配置。
- [x] **新增字段**：已执行 `DEFINE FIELD embedding_dashscope ON document_chunks TYPE array<float>;`。
- [x] **备份**：已执行 `CREATE document_chunks_backup AS SELECT * FROM document_chunks;`。
- [x] **重新向量化**：已运行 WHILE 循环脚本，将 `content` 通过 `text-embedding-v4` 生成向量写入 `embedding_dashscope`。
- [x] **验证**：已确认 `embedding_dashscope` 无 NONE 值，且能正常用于 `<|10,COSINE|>` 搜索。
- [x] **代码修改**：
    - [ ] `config.yaml` 模型名更新为 `text-embedding-v4`。
    - [ ] `settings.py` 配置类已更新。
    - [ ] `retrieval_client.py` 搜索 SQL 已更新为使用 `embedding_dashscope` 字段。
    - [ ] `vectorize.py` 向量化流水线已更新。

---

