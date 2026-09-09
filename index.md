# Wiki Index

> Content catalog. Every wiki page listed under its type with a one-line summary.
> Read this first to find relevant pages for any query.
> Last updated: 2026-09-07 | Total pages: 82

## Entities

- [[dashscope]] — 阿里云嵌入模型 API (text-embedding-v4)，1024 维，替代 Ollama bge-m3
- [[delta-lake]] — 存储架构框架，在对象存储上引入事务日志管理 Parquet 文件
- [[duckdb]] — 嵌入式 OLAP 数据库，向量化执行引擎，万物皆可直接查
- [[iceberg]] — Apache Iceberg 开放表格式规范，元数据树 + ACID 事务 + 时间旅行
- [[iceberg-rest-catalog]] — Iceberg REST Catalog HTTP 网关，管理表元数据指针
- [[oss]] — 阿里云对象存储服务，无限扩容、12个9持久性、分级存储
- [[polars]] — Rust 底层高性能 DataFrame 库，多核并行 + 懒惰模式
- [[pyiceberg]] — Iceberg 纯 Python 客户端，轻量级执行读写
- [[risingwave]] — 流数据库，常驻内存物化视图 + Iceberg Sink 落湖
- [[lakekeeper]] — 自建 Iceberg REST Catalog，标准协议，元数据存 PG
- [[skillforge]] — AI Agent 技能运行时环境，可插拔技能 + 记忆持久化
- [[deepagents]] — LangChain 深度任务 Agent 库，LangGraph 薄封装 + todo/子代理/虚拟文件系统
- [[surrealdb]] — 多模型数据库，支持向量搜索、全文检索、图数据库
- [[fjall]] — 纯 Rust 嵌入式 LSM-Tree KV 引擎，本地 NVMe 极致性能
- [[slatedb]] — 纯 Rust 云原生 LSM-Tree KV 引擎，真理源在 S3
- [[surrealkv]] — 纯 Rust 嵌入式 KV 引擎，严格 MVCC 事务 + 时间旅行
- [[openraft]] — Rust Raft 共识库，为 Fjall+Raft 架构提供分布式一致性

## Concepts

- [[agent-compound-interest]] — Agent 复利：持久化（Memory/Skills/Cron）让每次交互产生跨会话累积
- [[agent-cache-memory-optimization]] — Agent 四级优化：语义缓存/工具缓存/推理链缓存/分层记忆，压缩率 ~80%
- [[agent-memory]] — Agent 记忆两层架构：集成层（框架相关）+ 处理层（框架无关）
- [[agent-usage-patterns]] — Agent 使用模式：主动驾驶、负面指令、看 diff、纠正写入持久化
- [[automation-skill-workflow]] — 自动化 Skill 工作流：抓包录制→AI 编译→双模降级执行
- [[cap-theorem]] — 分布式系统 CAP 三要素：一致性、可用性、分区容错性不可兼得
- [[cdc-vs-api-sync]] — CDC binlog 无感捕捉 vs API 定时拉取，两种数据同步模式对比
- [[data-pipeline-workflow]] — 数据管道：DuckDB 查询 + Polars ETL + Delta Lake 写入 + Sync-on-Query
- [[ecommerce-search]] — 电商搜索：相关性门槛原则 + 准乘法融合 + 5步全流程
- [[embedding-migration]] — 嵌入模型从 Ollama 迁移到 DashScope 的双向量并行策略
- [[graph-memory]] — 图谱化记忆：原子三元组 + 三种聚簇策略 + 权重系统
- [[hnsw-index]] — HNSW 近似最近邻向量索引，O(N)→对数级搜索复杂度
- [[hybrid-search]] — 硬过滤+双路召回+业务重排的混合搜索架构
- [[iceberg-cow-problem]] — Iceberg Copy-on-Write 高频小批量更新的写放大和锁冲突
- [[iceberg-small-file-problem]] — 小数据量+过度分区导致元数据膨胀和性能坍塌
- [[kv-cache]] — KV Cache 与前缀缓存：checkpoint 不可变设计保证缓存持续命中
- [[lakehouse]] — 湖仓一体：表格式+对象存储+按需计算，替代专用集群
- [[lance-oss-compatibility]] — Lance 与 OSS 不兼容：条件写入协议差异，需先写本地再上传
- [[llm-fundamentals]] — LLM 基础认知：概率预测器、三局限、价值=视野×速度
- [[llm-overthinking]] — LLM 过度思考批判：跑分越高干活越差，厚 Harness 不需要深度 CoT
- [[llm-prompt-patterns]] — LLM 隐藏行为模式：换词比加约束有效，选模板而非对抗模型
- [[bm25]] — BM25 概率排序算法，TF-IDF 进化版，$k_1$ 词频饱和 + $b$ 长度归一化
- [[ngram-analyzer]] — l3gram 自定义分析器，ngram(1,3) 支持中文和模糊匹配
- [[polars-iceberg-oss-tables]] — Polars write_iceberg 与 OSS Tables 协议不兼容问题
- [[rrf-fusion]] — 倒数排名融合算法，只看排名不看分数
- [[search-linear]] — 线性加权融合函数，支持 Min-Max 归一化和自定义权重
- [[task-data-pipeline]] — OA 待办数据管道：MySQL CDC → RisingWave → Lakekeeper Iceberg → DuckDB 查询
- [[two-stage-search-pipeline]] — 两阶段流水线：向量召回 + 文本召回 + RRF 融合 + 四维业务重排
- [[composite-key-encoding]] — 复合键编码：Redis 数据结构在纯 KV 中的字节序模拟
- [[design-patterns]] — 纯 KV 四大设计模式：Index-Only Scan、Bitmap 拦截、应用层 MVCC、WiscKey
- [[okm-framework]] — OKM 对象-键空间映射：过程宏 + 数字命名空间，代码即 DDL
- [[application-mvcc]] — 应用层 MVCC：版本号编入 Key 实现无锁时间旅行
- [[wisckey-separation]] — WiscKey 键值分离：大 Value 场景的写放大解药
- [[raft-consensus]] — Raft 共识：Leader 选举 + 日志复制 + 状态机 Apply
- [[two-architecture-paths]] — 两条架构路径：Fjall+Raft（本地极致）vs SlateDB+S3（云原生）
- [[sql-vs-kv-pipeline]] — SQL vs KV 管道链：固定查询模式下的降维打击
- [[agent-memory-kv]] — Agent 记忆系统的 KV 落地：时间线扫描 + 精确点查 + 二级索引
- [[openresty-kv-gateway]] — OpenResty + KV 网关：Rust Sidecar 替代 Redis
- [[btree-vs-redblack-tree]] — B-Tree（磁盘矮胖子）vs 红黑树（内存瘦高个）
- [[storage-evolution-chain]] — 存储进化链：红黑树→B-Tree→LSM-Tree→开放表格式，四级跃迁驱动力
- [[ecs-entity-component-system]] — ECS 实体组件系统：标签（一物多标签）替代树（class 继承），数据库心智模型
- [[modern-language-design]] — 现代语言设计四层面：语法→语义→类型系统→抽象设施，统摄规律：复杂度守恒
- [[query-language-design]] — 查询语言设计：SQL/DataTable/SurQL/KV 四种数据交互参照系对比
- [[kdl-config-formats]] — KDL 节点式配置语言 vs JSON/TOML/YAML 哲学分野
- [[kv-advanced-encoding]] — KV 复合键编码高级技术：补码反转、双写原子性、哈希打散、长度前缀
- [[task-memory-ticket]] — 任务记忆（Ticket）：多步 SKILL 任务态隔离，参数持久化 + 惰性 gate + 多任务并存
- [[agent-status-bar]] — Agent 状态列：上下文末尾注入结构化元信息，防止 LLM 被闲聊带跑
- [[unified-data-layer]] — 统一数据层架构：SurrealDB 多模型统一替代 Polyglot Persistence，计算下推 + 反 ORM
- [[engineering-mindset]] — 工程思维与工程素养：工程师双核心能力，约束下求最优可行解

- [[bloom-filter]] — 布隆过滤器：空间效率极高的概率型数据结构，用于缓存穿透防护和黑名单过滤
- [[sse-server-sent-events]] — SSE（Server-Sent Events）：标准 HTTP 请求的升级，服务器分块持续推送数据
- [[websocket]] — WebSocket 协议：HTTP 握手后切换为独立帧协议，全双工通信
- [[http3-quic]] — HTTP/3 与 QUIC：基于 UDP 的用户态传输协议，解决 TCP 队头阻塞
- [[sql-injection-parameterized-queries]] — SQL 注入与参数化查询：占位符原理、AI 生成 SQL 的三层防护
- [[model-parameters-quantization]] — 模型参数量与 INT8/INT4 量化：参数量决定智商上限，量化决定运行门槛
- [[npu-vs-gpu]] — NPU vs GPU：NPU 在推理能效比碾压 GPU，GPU 在通用算力和大模型训练占统治地位
- [[traditional-vs-moe-models]] — 传统模型 vs MoE 模型：稠密模型全参数激活，MoE 路由网络动态指派专家
- [[emergent-skill]] — 涌现式 SKILL：技能从使用中自己长出来，图可达性替代 ACL，规模越大越划算
- [[prefix-checkpoint]] — Prefix Checkpoint：不可变 checkpoint 实现上下文压缩，对 KV cache 友好
- [[skill-direct-invocation]] — Skill 直接调用与推送：绕过 LLM 执行确定性操作，脚本主动推送前端
- [[skill-form-param-collection]] — Skill 表单化参数收集：缺参推 json-render 表单，用户填写带参回调，LLM 零参与

## Comparisons

- [[ollama-vs-dashscope]] — 本地 Ollama vs 云端 DashScope 嵌入方案对比
- [[embedded-kv-vs-redis]] — 嵌入式 KV vs Redis：TCO 1/8、延迟 1/16、运维 1/8
- [[skill-vs-framework]] — Skill 模式 vs 框架路线：编排智能放模型上下文 vs 放框架代码，光谱正在右移
- [[sync-on-query-vs-cdc]] — asw（Sync-on-Query 客户端写入 + OSS Tables）vs search-todo（CDC 流式 + Lakekeeper）：复杂度藏在代码里 vs 摆在 docker-compose 里

## Queries
