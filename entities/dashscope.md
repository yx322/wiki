---
title: DashScope (text-embedding-v4)
created: 2026-07-21
updated: 2026-07-21
type: entity
tags: [dashscope, embedding, vector-search]
sources: [raw/articles/SurrealDB_DashScope_Migration_Guide.md, raw/articles/SurrealDB向量集成方案分析.md, raw/articles/DashScope和ollama-bge-m3.md]
confidence: high
---

# DashScope (text-embedding-v4)

阿里云提供的嵌入模型 API，兼容 OpenAI 格式。在本项目中替代 Ollama bge-m3 作为向量生成方案。

## 关键参数

| 参数 | 值 |
|------|-----|
| 模型 | text-embedding-v4 |
| 维度 | 1024 |
| API 端点 | `https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings` |
| 输入格式 | `input: ["文本"]`（必须是数组） |
| 响应路径 | `data[0].embedding` |

## 在 SurrealDB 中的集成

通过自定义函数 `fn::dashscope_embed($text)` 封装 HTTP 调用：

```sql
DEFINE FUNCTION fn::dashscope_embed($text: string) {
    LET $config = SELECT value FROM config WHERE name = 'dashscope_api_key' LIMIT 1;
    LET $api_key = $config[0].value.api_key;
    IF !$api_key { THROW "Missing DashScope API Key"; };
    LET $resp = http::post(
        "https://dashscope.aliyuncs.com/compatible-mode/v1/embeddings",
        { model: "text-embedding-v4", input: [$text] },
        { Authorization: string::concat("Bearer ", $api_key), "Content-Type": "application/json" }
    );
    RETURN $resp.data[0].embedding;
};
```

## 从 Ollama 迁移

- 旧模型：Ollama bge-m3（本地部署，1024 维）
- 新模型：DashScope text-embedding-v4（云端 API，1024 维）
- **关键**：两者生成的向量在完全不同的坐标空间，不能混用
- 迁移策略：保留旧 `embedding` 字段，新增 `embedding_dashscope` 字段，批量刷新

## 批量刷向量脚本

```sql
WHILE (SELECT count() FROM table WHERE embedding_dashscope = NONE) > 0 {
    LET $chunks = SELECT id, content FROM table WHERE embedding_dashscope = NONE LIMIT 50;
    FOR $chunk IN $chunks {
        TRY {
            LET $vec = fn::dashscope_embed($chunk.content);
            UPDATE $chunk.id SET embedding_dashscope = $vec;
        } CATCH { /* skip failed */ };
    };
    SLEEP 0.1;  -- 避免 API 限流
};
```

## 参见
- [[surrealdb]] — 数据库集成
- [[ollama-vs-dashscope]] — 方案对比
- [[embedding-migration]] — 迁移指南

^[raw/articles/SurrealDB_DashScope_Migration_Guide.md]
^[raw/articles/SurrealDB向量集成方案分析.md]
