# 📚 个人 LLM Wiki 知识库

> 一次编译、持续维护的个人知识库。基于 [Karpathy 的 LLM Wiki 模式](https://karpathy.bearblog.dev/keep-a-wiki/) —— 不做检索式 RAG，而是把知识**主动整理成互相链接的 Markdown 页面**，让 LLM（和我自己）能读懂、能溯源、能持续生长。

[![Pages](https://img.shields.io/badge/pages-76-blue)](index.md) [![Mode](https://img.shields.io/badge/mode-wiki%20%2B%20raw-orange)]() [![Lang](https://img.shields.io/badge/language-%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87-red)]()

---

## 这是什么

一个围绕 **AI Agent / 存储系统 / 搜索技术** 构建的个人知识库，特点是：

- **wiki + raw 双层结构** — raw 存不可变的原始素材，wiki 存提炼后的知识页面
- **互相链接** — 页面之间用 `[[wikilinks]]` 交叉引用，形成知识网络
- **单一事实来源** — 每个概念一个页面，更新时改一处
- **可溯源** — 每页 frontmatter 标注 `sources`，随时能查到出处
- **Obsidian 友好** — 直接用 Obsidian 打开即可浏览，带关系图谱

## 目录结构

```
wiki/
├── index.md            # 总目录（按主题分组 + 页面索引）
├── log.md              # 操作日志（每次摄入/更新的记录）
├── SCHEMA.md           # 页面格式规范
├── entities/           # 实体页：具体项目/工具/系统（skillforge、Iceberg、SurrealDB…）
├── concepts/           # 概念页：技术原理/机制/模式（KV Cache、HNSW、RRF…）
├── comparisons/        # 对比页：X vs Y 型技术选型分析
├── queries/            # 查询页：常见问题的速查入口
├── raw/                # 原始素材（不可变，只增不改）
│   └── articles/       #   文档、对话整理、项目解析原文
└── .obsidian/          # Obsidian 配置
```

## 知识版图

### 🤖 AI Agent 与记忆

| 主题 | 核心页面 |
|------|---------|
| Agent 记忆架构 | [[agent-memory]] · [[agent-memory-kv]] · [[graph-memory]] |
| 记忆压缩 | [[prefix-checkpoint]] — 不可变 checkpoint，对 KV cache 友好 |
| 任务记忆 | [[task-memory-ticket]] — 两层表 + 惰性 gate |
| 涌现式技能 | [[emergent-skill]] — 技能从使用中长出来，图可达性替代 ACL |
| 技能直调 | [[skill-direct-invocation]] — 确定性操作绕过 LLM |
| KV Cache | [[kv-cache]] — LLM 推理加速的核心机制 |

### 🗄️ 存储系统

| 主题 | 核心页面 |
|------|---------|
| KV 存储引擎 | [[storage-evolution-chain]] · [[wisckey-separation]] · [[kv-advanced-encoding]] |
| 数据结构 | [[btree-vs-redblack-tree]] · [[composite-key-encoding]] · [[bloom-filter]] |
| 分布式共识 | [[raft-consensus]] · [[cap-theorem]] |
| 嵌入式 KV | entities: [[fjall]] · [[slatedb]] · [[surrealkv]] |

### 🌊 湖仓一体

| 主题 | 核心页面 |
|------|---------|
| Iceberg | [[iceberg]] · [[lakehouse]] · [[iceberg-cow-problem]] · [[iceberg-small-file-problem]] |
| 计算引擎 | [[duckdb]] · [[polars]] · [[polars-iceberg-oss-tables]] |
| 数据湖 | [[delta-lake]] · entities: [[oss]] · [[dashscope]] |

### 🔍 搜索技术

| 主题 | 核心页面 |
|------|---------|
| 混合检索 | [[hybrid-search]] · [[rrf-fusion]] · [[two-stage-search-pipeline]] |
| 分词与评分 | [[bm25]] · [[ngram-analyzer]] |
| 向量检索 | [[hnsw-index]] · [[search-linear]] |
| 电商实战 | [[ecommerce-search]] · [[sql-vs-kv-pipeline]] |

### ⚙️ 编程语言与设计

| 主题 | 核心页面 |
|------|---------|
| 范式 | [[ecs-entity-component-system]] — 标签 vs 树 |
| 语言设计 | [[modern-language-design]] · [[query-language-design]] · [[kdl-config-formats]] |
| 设计模式 | [[design-patterns]] · [[engineering-mindset]] |

> 完整索引见 **[index.md](index.md)**，当前共 **76 页**。

## Wiki vs RAG

| 维度 | 本 Wiki（编译式） | 传统 RAG（检索式） |
|------|-----------------|------------------|
| 知识组织 | 人工/LLM 主动提炼成页面 | 文档切块扔进向量库 |
| 一致性 | 单一来源，更新一处 | 同一知识可能散在多个 chunk |
| 上下文效率 | 直接注入相关页面，无噪音 | 检索结果良莠不齐 |
| 溯源 | frontmatter 记录 sources | 引用 chunk 位置 |
| 适合 | 稳定的核心知识、反复使用 | 海量长尾文档、一次性查询 |

## 如何使用

**用 Obsidian 打开（推荐）**：

```
git clone https://github.com/yx322/wiki.git
```
然后用 Obsidian 打开该目录，`[[wikilinks]]` 会变成可点击链接，关系图谱立即可见。

**当 LLM 的知识源**：
把 `index.md` + 相关概念页注入 system prompt，或在 Agent 框架里挂载本目录作为知识库 —— 页面都是自包含的 Markdown，无需预处理。

## 维护规则

1. **raw 不可变** — 原始素材只增不改，保证溯源可靠
2. **一概念一页** — 每个知识单元只有一个页面，其他地方用链接引用
3. **更新必记日志** — 每次摄入/修改在 `log.md` 留痕
4. **frontmatter 完整** — title / type / tags / sources / created / updated / confidence 缺一不可
5. **交叉引用优先** — 新页面生成时主动建立与既有页面的 `[[wikilinks]]`

格式规范详见 [SCHEMA.md](SCHEMA.md)。

---

*最后更新：2026-09 · 共 76 页 · 持续生长中* 🌱
