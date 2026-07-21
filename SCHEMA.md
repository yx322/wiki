# Wiki Schema

## Domain
数据工程与 AI 应用开发：涵盖数据湖仓架构（Iceberg/Delta Lake/OSS）、向量搜索引擎（SurrealDB/DashScope）、电商搜索与推荐系统、LLM Agent 工程实践（工具调用、记忆系统、上下文管理）。

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `surrealdb-hnsw-index.md`)
- 中文内容的页面也用英文文件名，内容用中文
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- **Provenance markers:** On pages that synthesize 3+ sources, append `^[raw/articles/source-file.md]` at the end of paragraphs whose claims come from a specific source.

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
confidence: high | medium | low
contested: true  # optional, set when unresolved contradictions exist
contradictions: [other-page-slug]  # optional
---
```

### raw/ Frontmatter
```yaml
---
source_url: https://example.com/article  # if applicable
ingested: YYYY-MM-DD
sha256: <hex digest of body content>
---
```

## Tag Taxonomy

### 数据存储与架构
- lakehouse — 湖仓一体化架构
- iceberg — Apache Iceberg
- delta-lake — Delta Lake
- oss — 对象存储 / 阿里云 OSS
- polars — Polars 数据处理

### 搜索与向量
- surrealdb — SurrealDB 数据库
- vector-search — 向量搜索 / HNSW
- embedding — 嵌入模型 / DashScope
- fulltext-search — 全文搜索
- hybrid-search — 混合搜索

### 电商与业务
- ecommerce — 电商系统
- recommendation — 推荐系统
- quotation — 报价系统
- product-search — 商品搜索

### LLM 与 Agent
- llm — 大语言模型基础
- agent — AI Agent 架构
- cot — 思维链 / Chain of Thought
- tool-calling — 工具调用
- memory — Agent 记忆系统
- rag — 检索增强生成
- kv-cache — KV Cache / 推理优化

### 嵌入式 KV 与分布式
- embedded-kv — 嵌入式 KV 存储引擎
- fjall — Fjall LSM-Tree 引擎
- slatedb — SlateDB 云原生引擎
- surrealkv — SurrealKV MVCC 引擎
- openraft — Openraft Raft 共识
- composite-key — 复合键编码
- lsm-tree — LSM-Tree 数据结构
- consensus — 分布式共识
- redis — Redis 替代方案批判

### 元标签
- comparison — 方案对比
- migration — 迁移指南
- troubleshooting — 问题排查
- architecture — 系统设计
- performance — 性能优化

Rule: every tag on a page must appear in this taxonomy. If a new tag is needed, add it here first.

## Page Thresholds
- **Create a page** when an entity/concept appears in 2+ sources OR is central to one source
- **Add to existing page** when a source mentions something already covered
- **DON'T create a page** for passing mentions or things outside the domain
- **Split a page** when it exceeds ~200 lines
- **Archive a page** when content is fully superseded

## Entity Pages
One page per notable entity (SurrealDB, DashScope, Iceberg, Polars, etc.). Include:
- Overview / what it is
- Key facts, versions, capabilities
- Relationships to other entities ([[wikilinks]])
- Source references

## Concept Pages
One page per concept (HNSW, KV Cache, 湖仓一体, etc.). Include:
- Definition / explanation
- Current state of knowledge
- Open questions or debates
- Related concepts ([[wikilinks]])

## Comparison Pages
Side-by-side analyses. Include:
- What is being compared and why
- Dimensions of comparison (table format preferred)
- Verdict or synthesis
- Sources

## Update Policy
When new information conflicts with existing content:
1. Check dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in lint report
