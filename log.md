# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> When this file exceeds 500 entries, rotate: rename to log-YYYY.md, start fresh.

## [2026-07-21] create | Wiki initialized
- Domain: 数据工程与 AI 应用开发（湖仓架构、向量搜索、电商系统、LLM Agent）
- Location: D:\wiki
- Structure created with SCHEMA.md, index.md, log.md
- Raw sources available at D:\md (24 files + 3 subdirectories)

## [2026-07-21] ingest | SurrealDB 批次（6篇源文档）
- Raw: 5 files copied to raw/articles/
- Created: entities/surrealdb.md, entities/dashscope.md
- Created: concepts/hnsw-index.md, concepts/hybrid-search.md, concepts/ngram-analyzer.md, concepts/rrf-fusion.md, concepts/search-linear.md, concepts/embedding-migration.md
- Created: comparisons/ollama-vs-dashscope.md
- Total new pages: 9

## [2026-07-21] ingest | 湖仓/Iceberg/OSS 批次（9篇源文档）
- Raw: 9 files copied to raw/articles/
- Created: entities/iceberg.md, entities/delta-lake.md, entities/duckdb.md, entities/polars.md, entities/risingwave.md, entities/oss.md, entities/pyiceberg.md, entities/iceberg-rest-catalog.md
- Created: concepts/lakehouse.md, concepts/iceberg-small-file-problem.md, concepts/iceberg-cow-problem.md, concepts/cap-theorem.md, concepts/cdc-vs-api-sync.md, concepts/polars-iceberg-oss-tables.md
- Total new pages: 14

## [2026-07-21] ingest | LLM/Agent 批次（10篇源文档）
- Raw: 10 files copied to raw/articles/
- Created: concepts/llm-fundamentals.md, concepts/llm-prompt-patterns.md, concepts/llm-overthinking.md, concepts/agent-compound-interest.md, concepts/agent-memory.md, concepts/graph-memory.md, concepts/kv-cache.md, concepts/agent-usage-patterns.md
- Total new pages: 8

## [2026-07-21] ingest | 综合批次（7篇源文档）
- Raw: 7 files copied to raw/articles/
- Created: concepts/ecommerce-search.md, concepts/automation-skill-workflow.md, concepts/data-pipeline-workflow.md, concepts/lance-oss-compatibility.md, concepts/task-data-pipeline.md
- Created: entities/skillforge.md
- Total new pages: 6

## [2026-07-21] ingest | KV 存储引擎批次（2篇源文档）
- Raw: kv-storage-engine.md (948 lines), object-keyspace-mapping.md (656 lines)
- Created: entities/fjall.md, entities/slatedb.md, entities/surrealkv.md, entities/openraft.md
- Created: concepts/composite-key-encoding.md, concepts/design-patterns.md, concepts/okm-framework.md, concepts/application-mvcc.md, concepts/wisckey-separation.md, concepts/raft-consensus.md, concepts/two-architecture-paths.md, concepts/sql-vs-kv-pipeline.md, concepts/agent-memory-kv.md, concepts/openresty-kv-gateway.md
- Created: comparisons/embedded-kv-vs-redis.md
- Updated: SCHEMA.md (新增嵌入式 KV 标签体系)
- Total new pages: 15

## [2026-07-21] update | Delta Lake 物理存储结构补充
- Updated: entities/delta-lake.md (新增"简单理解"、"物理存储结构"、"与直接存 Parquet 的区别"、"数据查询链路"四个章节)
- Added cross-references: [[oss]]、[[duckdb]]
- Sources: 飞书对话（2026-07-21）

## [2026-07-21] create | B-Tree vs 红黑树
- Created: concepts/btree-vs-redblack-tree.md
- 红黑树=内存瘦高个（指针瞬移），B-Tree=磁盘矮胖子（打包数据压树高）
- Cross-references: fjall, sql-vs-kv-pipeline, composite-key-encoding

## [2026-07-21] update | B-Tree vs 红黑树：磁盘 IO 深度解释
- Updated: concepts/btree-vs-redblack-tree.md（新增"什么是磁盘 IO"、"磁盘 IO 到底有多慢"、"为什么磁盘 IO 这么慢"、"为什么 B-Tree 要矮胖"四个章节）
- Added: 延迟对比表（L1 缓存→HDD，纳秒到毫秒的 6 个数量级差距）
- Added: 10 亿数据查找的数学推导（红黑树 30 次 IO vs B-Tree 3 次 IO）
- Updated: D:\md\btree-vs-redblack-tree.md（同步扩展内容）

## [2026-07-21] create | 存储进化链
- Created: concepts/storage-evolution-chain.md
- 红黑树→B-Tree→LSM-Tree→开放表格式，四级跃迁驱动力
- Raw: D:\md\storage-evolution-chain.md
- Updated: concepts/btree-vs-redblack-tree.md（交叉引用）
- Updated: index.md

## [2026-07-21] ingest | 两阶段搜索流水线（1篇源文档）
- Raw: search-engine-two-stage-pipeline.md
- Created: concepts/two-stage-search-pipeline.md
- Updated: hybrid-search.md, rrf-fusion.md, hnsw-index.md, ecommerce-search.md (交叉引用)
- Total new pages: 1

## [2026-07-21] update | RRF 深度补充（传统归一化失效 + 三大算法优势）
- Updated: concepts/two-stage-search-pipeline.md (新增传统归一化失效分析 + RRF 三大归一化优势 + 五大算法优势扩展为 a~e)
- Updated: concepts/rrf-fusion.md (新增传统归一化失效分析 + 三大纯算法优势 + 其他算法优势)
- Sources: raw/articles/search-engine-two-stage-pipeline.md (追加内容)

## [2026-07-21] ingest | N-gram 与 BM25 对话记录（1篇源文档）
- Raw: ngram-bm25-analysis.md
- Created: concepts/bm25.md (BM25 概率排序算法完整解析)
- Updated: index.md (新增 BM25 条目)
- Total new pages: 1

## [2026-08-13] ingest | 语言设计与 ECS 批次（5篇源文档）
- Raw: entity-component-system.md, modern-language-design.md, query-language-design.md, kv-storage-engine.md (补充), kdl-vs-config-formats.md
- Created: concepts/ecs-entity-component-system.md, concepts/modern-language-design.md, concepts/query-language-design.md, concepts/kdl-config-formats.md, concepts/kv-advanced-encoding.md
- Note: kv-storage-engine.md 已在 KV 批次完整摄入，本次补充高级编码技术页面
- Total new pages: 5

## [2026-08-18] ingest | ADR-0005 任务记忆讨论记录
- Raw: adr-0005-task-memory.md (D:\md\ADR-0005-任务记忆讨论记录.md)
- Created: concepts/task-memory-ticket.md (任务记忆/Ticket：多步 SKILL 任务态隔离)
- Created: concepts/agent-status-bar.md (Agent 状态列：上下文末尾注入元信息，防闲聊带跑)
- Cross-references: agent-memory, kv-cache, skillforge, llm-fundamentals
- Total new pages: 2

## [2026-08-20] ingest | SQL 注入与参数化查询（1篇源文档）
- Raw: sql-injection-parameterized-queries.md (D:\md\sql-injection-parameterized-queries.md)
- Created: concepts/sql-injection-parameterized-queries.md (SQL 注入原理、占位符、AI 生成 SQL 安全防护)
- Sources: order-analytics 实际代码分析
- Total new pages: 1

## [2026-08-20] ingest | 模型参数量与量化（1篇源文档）
- Raw: model-parameters-quantization.md (D:\md\model-parameters-quantization.md)
- Created: concepts/model-parameters-quantization.md (参数量决定智商上限，量化决定运行门槛)
- Total new pages: 1

## [2026-08-20] ingest | NPU vs GPU 对比（1篇源文档）
- Raw: npu-vs-gpu.md (D:\md\npu-vs-gpu.md)
- Created: concepts/npu-vs-gpu.md (NPU 推理能效比碾压 GPU，GPU 通用算力和大模型训练占统治地位)
- Total new pages: 1

## [2026-08-20] ingest | 传统模型 vs MoE 模型（1篇源文档）
- Raw: traditional-vs-moe-models.md (D:\md\traditional-vs-moe-models.md)
- Created: concepts/traditional-vs-moe-models.md (稠密模型全参数激活，MoE 路由网络动态指派专家)
- Total new pages: 1

## [2026-08-20] ingest | SkillForge 项目解析（8篇源文档）
- Raw: skillforge-overview.md, skillforge-memory-system.md, skillforge-direct-invocation.md, skillforge-emergent-skills.md, skillforge-module-map.md
- Created: concepts/emergent-skill.md (涌现式 SKILL：图可达性替代 ACL)
- Created: concepts/prefix-checkpoint.md (不可变 checkpoint + KV cache)
- Created: concepts/skill-direct-invocation.md (直接调用 + 推送机制)
- Updated: entities/skillforge.md (交叉引用)
- Output: D:\md\skillforge-analysis\ (8篇整理文档)
- Total new pages: 3

## [2026-08-18] ingest | 网络协议深度解析（1篇源文档）
- Raw: network-protocols-deep-dive.md (D:\md\网络协议深度解析-HTTP-SSE-WebSocket-HTTP3.md)
- Created: concepts/sse-server-sent-events.md (SSE 服务器推送事件)
- Created: concepts/websocket.md (WebSocket 协议)
- Created: concepts/http3-quic.md (HTTP/3 与 QUIC)
- Created: concepts/bloom-filter.md (布隆过滤器)
- Total new pages: 4

## [2026-08-18] ingest | 剩余文件处理（3篇源文档）
- Raw: unified-data-layer.md, engineering-mindset-and-competence.md
- Created: concepts/unified-data-layer.md (统一数据层架构：SurrealDB 多模型统一)
- Created: concepts/engineering-mindset.md (工程思维与工程素养)
- Skipped: 解释 Skillforge 文件.md (内容已被 entities/skillforge.md 覆盖)
- Total new pages: 2

## [2026-09-03] ingest | MySQL CDC → RisingWave → Iceberg (Lakekeeper + OSS) 对话记录（1篇源文档）
- Raw: cdc-risingwave-iceberg-lakekeeper-dialog.md (D:\md\cdc-risingwave-iceberg-lakekeeper-对话记录.md)
- Created: entities/lakekeeper.md (自建 Iceberg REST Catalog，标准协议，元数据存 PG，server 注册进 Warehouse 是最常见坑)
- Created: comparisons/sync-on-query-vs-cdc.md (asw Sync-on-Query 客户端写入 vs search-todo CDC 流式：复杂度藏在代码里 vs 摆在 docker-compose 里)
- Updated: entities/iceberg-rest-catalog.md (+参见 lakekeeper)
- Updated: entities/risingwave.md (+参见 lakekeeper / sync-on-query-vs-cdc)
- Updated: concepts/task-data-pipeline.md (+参见 lakekeeper / sync-on-query-vs-cdc)
- Total pages: 78

## [2026-09-07] ingest | DeepAgent 基于 LangChain 新功能对话记录（1篇源文档，10轮Q&A）
- Raw: deepagent-langchain-dialog.md (D:\md\langgraph-Deepagent-skill-compare\DeepAgent基于LangChain新功能-Zai对话记录.md)
- Created: entities/deepagents.md (LangGraph 薄封装，todo/子代理/虚拟文件系统，编排框架≠推理引擎)
- Created: comparisons/skill-vs-framework.md (编排智能放模型上下文 vs 放框架代码，光谱右移，skillforge 定位论证)
- Created: concepts/agent-cache-memory-optimization.md (语义缓存/工具缓存/推理链缓存/分层记忆四级优化，压缩率~80%)
- Updated: concepts/emergent-skill.md (+参见 skill-vs-framework / deepagents)
- Total pages: 81

## [2026-09-07] ingest | Skill 表单化参数收集方案对话记录（1篇源文档）
- Raw: skillforge-form-param-collection-dialog.md (D:\md\skillforge-analysis\09-表单化参数收集方案-json-render对话记录.md)
- Created: concepts/skill-form-param-collection.md (缺参推 json-render 表单 + hidden 带参回调，LLM 零参与，无状态设计)
- Updated: index.md (Total pages: 82)
- 状态：方案确认完毕待实施（push_form + typer 内省生成 schema + calls.py 兜底校验）
