# 记忆架构设计

## 一、两层架构

记忆系统分为**集成层**和**处理层**。集成层与 Agent 框架耦合，处理层框架无关。

```
┌─────────────────────────────────┐
│         集成层（框架相关）         │
│  触发时机 │ 注入方式 │ 框架适配   │
├─────────────────────────────────┤
│         处理层（框架无关）         │
│  提取     │ 存储     │ 检索      │
└─────────────────────────────────┘
```

| 层 | 职责 | 演进时改什么 |
|:--|:--|:--|
| **集成层** | 什么时候捕获、怎么注入 prompt、怎么适配 Agent 框架 | 换框架时改 |
| **处理层** | 对话→记忆的转换、存储格式、检索算法 | 换存储/检索策略时改 |

分层的价值：**现有方案（agentmemory、cognee）的组件耦合导致换任何一层都要动其他层**。分层后每层可独立演进——从 KV 迁移到图存储只改处理层，换框架只改集成层。

### 集成层

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **触发时机** | auto hooks（每轮系统自动） / 手动调用（Agent 判断） / 阈值触发 | agentmemory 用 hooks，skillforge 用阈值 |
| **注入方式** | 全量注入 / top-K 检索注入 / checkpoint + 增量 | Hermes 全量，agentmemory top-K，skillforge checkpoint |
| **框架适配** | 适配层接口（get_session_messages / add_message） | SnapshotSessionDB 适配 Agno |

### 处理层

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **提取** | flat KV（content + category + tags） / graph triples（subject + predicate + object） | skillforge 当前用 flat，图谱化用 triples |
| **存储** | SQLite / SurrealDB / Postgres / 三存储分离 | agentmemory 用 SQLite，skillforge 用 SurrealDB |
| **检索** | 向量 / BM25 / 图遍历 / RRF 融合 / 多策略路由 | agentmemory 向量+关键词，skillforge HNSW+BM25+RRF，cognee 9 种策略 |

### 计算时机光谱

处理层的核心区别在**计算发生在哪个阶段**：

```
写时重 ─────────────────────────────────── 读时重

LLM Wiki        KG (属性图)       KV 模式         向量模式
                graph-memory      skillforge      agentmemory
                SurrealDB graph   SurrealDB KV    SQLite + vector

写: LLM 整理/嵌入/拓扑推断    写: 简单切分/存储
读: 直接载入/遍历              读: 匹配/排序/向量搜索
```

光谱位置取决于使用方式，不取决于存储产品。SurrealDB 的多模型架构覆盖全部区间。

---

## 二、集成层设计

### 触发时机

| 方式 | 机制 | 优点 | 缺点 |
|:--|:--|:--|:--|
| **auto hooks** | 系统挂载在 Agent 生命周期上，每轮自动捕获 | 零侵入，不遗漏 | 捕获冗余数据（Agent 不会主动记的东西大多也不值得记） |
| **手动调用** | Agent 判断"这条值得记住"时调 memory_store | 精准，只存有价值的 | 依赖 Agent 判断力，可能遗漏 |
| **阈值触发** | 消息累积到 N 条时 LLM 批量提取 | 低频调用，LLM 开销可控 | N 条内未压缩（但原始消息仍在 session 中，Agent 可直接读取） |

auto hooks 的实际影响有限：捕获的内容（tool call 参数/返回值、error traceback）本身就在 session 消息里，hooks 只是把 session 里已有的东西又存了一份。真正区别是 session 关闭后的持久化——WorkingMemory 的 session_messages 表已解决此问题。

### 注入方式

| 方式 | 机制 | cache 友好度 |
|:--|:--|:--|
| **全量注入** | 每轮把所有历史注入 prompt | 差（内容每轮变化，KV cache 失效） |
| **top-K 检索** | 每轮检索相关记忆注入 | 中（检索结果可能变化） |
| **checkpoint + 增量** | 不可变 checkpoint + 最近 N 条消息 | 高（checkpoint 固定，可永久缓存） |

#### checkpoint + 增量机制

类似 event-sourcing 的快照模式。消息逐条写入 session_messages 表，达到阈值时压缩为 checkpoint（写入 session_checkpoints 表），后续所有未压缩消息即为增量：

```
msg_001 ... msg_100    ← 消息追加到 session_messages 表
              ↓ 达到阈值（100 条）
ckpt_0: summary="用户讨论了项目架构，决定用 SurrealDB..."
              ↓ 写入 session_checkpoints 表，不可变
msg_101 ... msg_150    ← 全部是增量（当前所有未压缩消息）
              ↓ 再次达到阈值
ckpt_1: summary="..."
msg_151 ...            ← 新的增量
```

**读取流程**：

消息一条条追加到 prompt 中（不是每次重新拼接）。LLM API 的多轮对话机制天然缓存前面所有 tokens：

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← 前面的 tokens 走 KV cache
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: 达到阈值 → 压缩为 ckpt_1 → 重置
Turn N+1: [ckpt_1] + [msg_201]               ← 新 checkpoint，重新开始
```

**写入流程**：
1. 每条消息追加到 session_messages 表
2. 检查距上次 checkpoint 的消息数是否 >= 阈值
3. 达到阈值 → 在当前 prompt 末尾追加压缩指令（充分利用已缓存的上下文）→ LLM 输出 checkpoint 摘要 + 长期记忆 → 指令本身不写入 session，输出写入 checkpoint 表，增量归零

**压缩时的缓存利用**：阈值到达时，当前 prompt 已包含 `[ckpt] + [msg_101..msg_150]`，全部在 KV cache 中。此时在末尾追加压缩指令（如"分析以上对话，输出 checkpoint 和 memories"），LLM 以缓存的完整上下文处理这条新指令，成本极低。指令是临时的——输出保存后丢弃，不写入 session，不污染后续对话。

**为什么 cache 友好**：checkpoint 一旦写入不可变，后续所有 turn 共享同一个 checkpoint 文本。LLM API 的 prompt caching 将 checkpoint 部分缓存在 GPU 显存中，只有增量部分每次变化。随着增量消息增多、下一次 checkpoint 触发，增量被压缩为新的固定摘要，缓存再次命中。

**与 Agno 滑动窗口的对比**：Agno 的 `add_history_to_context` 每次从 DB 读最近 N 条完整消息。随着新消息到来，N 条的组成不断变化（旧的被挤出、新的被加入），KV cache 每轮失效。checkpoint 模式下，历史被压缩为固定摘要，只有未压缩的增量部分变化，cache 持续命中。

### 框架适配

适配层隔离框架差异，核心接口只有两个方法：

```python
class SessionAdapter:
    def get_session_messages(session_id) → list[dict]  # checkpoint + 所有增量
    def add_message(session_id, message)                 # 追加消息
```

`get_session_messages` 返回 checkpoint + 后续所有未压缩消息，长度由阈值决定，不需要 limit 参数。

换框架时只重写适配层（几十行），处理层完全复用。

---

## 三、处理层设计

### 提取

从对话到记忆的转换，两种格式：

| 格式 | 结构 | 适用场景 |
|:--|:--|:--|
| **flat KV** | content + category + importance + tags | 简单偏好/事实，当前实现 |
| **graph triples** | subject + predicate + object + category | 有关系结构的知识，图谱化演进方向 |

两种格式可以在同一次 LLM 调用中同时输出（双层输出）：

```
100 条消息 → [LLM 单次调用]
                ├→ checkpoint 摘要（注入 prompt）
                └→ flat memories / graph triples（写入存储）
```

#### 提取时机：两种方案

**方案 A：每轮并行提取（⚠️ 过时）**

在 system prompt 中指示 LLM 同时输出用户回复和抽取三元组的函数调用，一次响应完成两件事。

问题：(1) 需要改造 Agent 框架；(2) 函数调用干扰 LLM 对用户回复的注意力；(3) 每轮引入新的函数调用 token，KV cache 命中率低。→ 详见 [图谱化记忆](graph-memory.md) §并行提取机制

**方案 B：阈值时末尾追加指令（当前方案）**

压缩指令在阈值到达时追加到 prompt 末尾，复用已缓存的完整上下文，指令本身不写入 session：

```
Turn 1..N: 正常对话，[ckpt] + 增量逐条追加，KV cache 持续命中
              ↓ 达到阈值
Turn N+1:  在 prompt 末尾追加 "分析以上对话，输出 checkpoint 和 memories"
           → LLM 以缓存上下文处理指令，输出写入存储，指令丢弃
Turn N+2:  新 checkpoint + 新增量，重新开始
```

优势：(1) 不改造 Agent；(2) 不干扰正常回复的注意力；(3) 利用缓存，边际成本低。

### 存储

| 后端 | 特点 | 适用 |
|:--|:--|:--|
| **SQLite** | 嵌入式，零部署 | 个人工具、本地 Agent |
| **SurrealDB** | KV + 向量 + 全文 + 图，多模型 | 企业服务、多用户 |
| **Postgres** | pgvector + SQL + graph backend | 已有 PG 基础设施 |

### 检索

| 算法 | 机制 | 确定性 |
|:--|:--|:--|
| **向量（HNSW）** | 语义相似度 | 概率性 |
| **BM25** | 关键词匹配 | 确定性 |
| **图遍历** | 实体关系链 | 确定性 |
| **RRF 融合** | 多路排序融合 | — |

RRF（Reciprocal Rank Fusion）是融合多路检索结果的标准做法——各路独立排序，按排名倒数加权合并。不依赖分数归一化，鲁棒性强。

---

## 四、skillforge 实现

### 当前方案（Phase 2.5）

在光谱中间偏右——比 agentmemory 检索质量高、cache 友好，比 cognee 部署轻、LLM 成本低。

```
写时重 ─────────────────────────────────── 读时重

cognee          skillforge        agentmemory
(知识图谱)       (当前实现)         (轨迹压缩)
SurrealDB graph  SurrealDB KV      SQLite + vector
```

#### 集成层

| 决策点 | 选择 |
|:--|:--|
| 触发时机 | 阈值触发（100 条）+ Agent 手动调用 |
| 注入方式 | checkpoint + 增量（SnapshotSessionDB） |
| 框架适配 | SnapshotSessionDB（适配 Agno session_db） |

#### 处理层

| 决策点 | 选择 |
|:--|:--|
| 提取 | flat KV（LLM 双层输出：checkpoint + memories） |
| 存储 | SurrealDB（session_messages + session_checkpoints + memories） |
| 检索 | HNSW + BM25 + RRF 三路融合 |

#### 核心接口

三个：`store` / `search` / `forget`。辅助接口 `list_all`（导出/备份）和 `get_stats`（仪表盘/运维）不参与 Agent 对话流程。

| 方法 | 用途 |
|:--|:--|
| `store(content, category, importance, tags)` | 存储记忆（embedding 由 SurrealDB 内部生成） |
| `search(query, top_k)` | 混合检索 |
| `forget(memory_id)` | 删除记忆 |

#### 关键设计决策

| 决策 | 原因 |
|:--|:--|
| 100 条以内 LLM 无感知 | 不膨胀 prompt，不破坏 KV cache |
| Checkpoint 不可变 | 写入后固定，可永久缓存 |
| 不依赖框架 MemoryManager | 黑盒提取不可控、框架绑定、每 turn 额外 LLM 调用 |
| Embedding 在 SurrealDB 内部生成 | Python 侧零传输零存储 |
| 不做过期清理 | 存储不是瓶颈，向量检索天然让不相关旧记忆排在后面 |

#### 框架迁移

SnapshotSessionDB 是唯一与框架耦合的部分（实现 `get_session_messages` + `add_message`）。MemoryManager 和 MemoryTools 完全复用。迁移成本：几十行适配代码。

---

## 五、图谱化演进

处理层的演进方向：提取从 flat KV 升级为 graph triples，检索增加图遍历。集成层不变。

```
当前：  flat KV + 向量/BM25/RRF
        ↓
演进：  graph triples + 向量/图遍历/RRF
```

### 提取变化

LLM 双层输出的第二层从 flat memories 变为 graph triples：

```
当前输出：
  memories: [{"content": "...", "category": "fact", "importance": 3, "tags": [...]}]

图谱化输出：
  triples: [{"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}]
```

一次 LLM 调用双层输出的模式不变，只是第二层的输出格式从 flat 变成 structured。

### 检索变化

在现有 HNSW + BM25 + RRF 基础上增加图遍历：

```
当前：query → 向量检索 + BM25 → RRF 融合
演进：query → 向量检索 + BM25 + 图遍历（多跳） → RRF 融合
```

### SurrealDB 的支撑

SurrealDB 的多模型架构让迁移自然——graph record 和 KV record 共存，检索时根据 query 类型路由。不需要换存储引擎。

### 详细设计

图谱化的完整设计（聚簇策略、权重系统、双层模型）见 [图谱化记忆](graph-memory.md)。

---

## 六、外部方案对比

### agentmemory（24k★，TypeScript + Rust）

**定位**：跨 Agent 的持久记忆层，通过 MCP 适配商业 IDE。

| 维度 | 实现 |
|:--|:--|
| 存储 | SQLite + 向量索引（`all-MiniLM-L6-v2` 本地 embedding） |
| 触发 | 12 个 auto hooks（每轮自动捕获） |
| 检索 | 向量 + 关键词（二元） |
| 注入 | 按需检索注入（92% token 节省） |

**优势**：MCP 通用性（任何支持 MCP/hook 的 Agent 都能接入）；auto hooks 零侵入。

**局限**：绑定 MCP 协议（端口常驻）；embedding 质量上限（小模型）；捕获内容本身已在 session 里（hooks 的增量价值有限）；无图结构、无权重系统。

### cognee（~22k★，Python）

**定位**：知识图谱 memory platform，ingest 任意格式数据构建 knowledge graph。

| 维度 | 实现 |
|:--|:--|
| 存储 | 三存储分离（relational + vector + graph） |
| 触发 | 手动 remember() |
| 检索 | 9 种策略路由（graph / vector / lexical / temporal / cypher 等） |
| 提取 | LLM 逐 chunk 提取 entity + relationship |

**优势**：图原生（多跳推理）；多策略检索路由；Postgres 统一栈（v1.0）；`improve()` + feedback_weight 自我改进。

**局限**：LLM 成本重（每 chunk 一次调用）；部署复杂（三存储）；无事实级冲突解决（旧事实和新事实并存）；无组织层（聚簇预聚合，靠 LLM 在检索时临时拼凑）。

### 对比矩阵

| 维度 | agentmemory | cognee | skillforge |
|:--|:--|:--|:--|
| **触发** | auto hooks（每轮） | 手动 remember() | 阈值触发 + 手动调用 |
| **提取** | iii-engine 压缩（黑盒） | LLM entity extraction | LLM 双层输出 |
| **存储** | SQLite + 向量 | 三存储分离 | SurrealDB（KV + 向量 + 全文） |
| **检索** | 向量 + 关键词 | 9 种策略路由 | HNSW + BM25 + RRF |
| **注入** | 按需检索 | GRAPH_COMPLETION 调 LLM | checkpoint + 增量 |
| **cache 友好** | 差（每轮变化） | 差（调 LLM） | 高（checkpoint 固定） |
| **框架绑定** | MCP | Python SDK + MCP | 无（适配层隔离） |
| **LLM 开销** | 低（压缩可配置） | 高（ingest 每 chunk） | 低（100 条才调一次） |

---

## 七、参考

### CodeGraph：代码结构层

本地语义代码知识图谱，代码修改时自动同步图谱，AI 查询时直接遍历。在光谱中属于"写时重"——inotify 触发自动建图，不需要人工维护。

与记忆系统互补：CodeGraph 回答"代码怎么组织的"（what），设计文档回答"为什么这样组织"（why），记忆系统回答"用这段代码学到了什么"（experience）。

### 纯记忆层技术全景（2026 开源）

四象限拓扑：

```
[象限 I：痕迹捕获]          [象限 II：情景事件/时序衰减]
  agentmemory (iii-engine)    Mnemosyne (Rust, Ebbinghaus)
  职责：捕获-压缩-检索闭环     HippoRAG (PPR 多跳联想)
───────┼──────────────────────────┼──────────────────
       │                          │
[象限 III：语义事实/冲突去重]   [象限 IV：向量/图谱引擎底座]
  Mem0 (实体-关系图谱)          sqlite-vec (纯 C, 单文件)
  cognee (知识图谱+多策略检索)  KuzuDB (嵌入式图数据库)
  职责：长效事实维护             职责：嵌入式免部署存储
```

### MCP 与记忆层

agentmemory 和 cognee 通过 MCP 适配商业 IDE，是"政治性妥协"。记忆层走 MCP 可接受（低频操作），工具执行层走 MCP 不可接受（高频操作）。skillforge 用框架适配层（SnapshotSessionDB）替代 MCP，进程内调用无网络开销。

### 合成闭环的缺口

理想的记忆闭环：捕获 → 压缩 → 冲突去重 → 持久化 → 技能固化。

**关键断裂：验证层缺失。** 每一步都可能引入错误——压缩丢失上下文、反思误判事实、冲突检测漏判。没有验证层，错误累积不是复利，是复亏。agentmemory 用 hooks 解决触发但绑定 MCP；cognee 用 improve() 部分解决进化但 LLM 成本重；skillforge 用人工审查保证质量但牺牲自动化。三难尚未被开源方案真正解决。

---

## 八、总结

### 当前状态

skillforge 已实现两层架构的集成层（阈值触发 + checkpoint 注入 + SnapshotSessionDB 适配）和处理层的基础形态（flat KV 提取 + HNSW/BM25/RRF 检索 + SurrealDB 存储）。核心接口三个：store / search / forget。

### 演进方向

| 阶段 | 改什么 | 不改什么 |
|:--|:--|:--|
| **图谱化** | 处理层：flat KV → graph triples，检索增加图遍历 | 集成层（触发、注入、适配）不动 |
| **框架迁移** | 集成层：重写 SnapshotSessionDB（几十行） | 处理层（MemoryManager、MemoryTools）不动 |
| **权重系统** | 处理层：增加 read_count/write_count 追踪和排序 | 集成层不动 |

### 设计原则

1. **集成层和处理层分离**：换框架只改集成层，换存储/检索只改处理层
2. **LLM 调用最小化**：100 条以内无感知，压缩时利用缓存末尾追加指令
3. **框架无关**：记忆逻辑不依赖任何特定 Agent 框架
4. **渐进式演进**：当前 flat KV 够用就用 flat KV，需要图结构时再升级
