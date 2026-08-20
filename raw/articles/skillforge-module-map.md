# SkillForge 项目解析：一句话看懂每个模块

---

## 项目一句话

**SkillForge = AI Agent 技能运行时**，让 LLM 通过自然语言调用你写好的业务技能（Skills），同时管理对话记忆、长期记忆和任务状态。

## 核心架构图

```
用户（微信/Web/CLI）
    ↓
┌──────────────────────────────────────────────────────────────┐
│  FastAPI Gateway（SSE + WebSocket + HTTP 直调）               │
│    ├─ /v1/chat/completions (SSE 流式聊天)                    │
│    ├─ /ws/chat (WebSocket 聊天 + 确定性调用)                  │
│    ├─ /skills/{name} (HTTP 直调 Skill 脚本)                   │
│    └─ /push (脚本 → 前端主动推送)                              │
├──────────────────────────────────────────────────────────────┤
│  认证层（Token 提取 → User API 验证 → ContextVar 注入）       │
├──────────────────────────────────────────────────────────────┤
│  Agent 引擎层                                                │
│    ├─ Agno 引擎（默认）：LocalSkills + 事件流转换              │
│    ├─ Hermes 引擎：OpenAI SDK + prompt 注入技能目录            │
│    └─ 自研引擎（规划中）：openai SDK 直连 while 循环           │
├──────────────────────────────────────────────────────────────┤
│  记忆系统（PostgreSQL）                                       │
│    ├─ 对话记忆：session_messages + session_checkpoints        │
│    ├─ 长期记忆：memories（HNSW + BM25 + RRF）                │
│    ├─ 任务记忆：tasks + task_items（两层表 + gate）           │
│    └─ 会话记忆：agno_sessions                                 │
├──────────────────────────────────────────────────────────────┤
│  Skill 执行层                                                │
│    ├─ Agent 路径：LLM 决定调用 → subprocess 执行 run.py      │
│    ├─ 直调路径：HTTP/WS → script_executor → run.py           │
│    └─ 环境变量注入：ContextVar → subprocess.env (CONTEXT_*)   │
└──────────────────────────────────────────────────────────────┘
```

## 每个模块干什么

| 文件/模块 | 一句话 |
|----------|--------|
| `cli.py` | 命令行入口：ask（单次问）、chat（交互）、serve（启服务）、init（建项目） |
| `config.py` | 所有配置定义，pydantic-settings，支持 TOML + 环境变量覆盖 |
| `context.py` | AgentContext：user_id + session_id + metadata（cookies/headers 等） |
| `core.py` | Agent 创建入口 + skills 注册表 + subprocess 拦截注入环境变量 |
| `events.py` | 流式事件基类（引擎无关）：RunContent/ToolCall/RunCompleted |
| `engine/agno.py` | Agno 引擎：Agent + LocalSkills + 事件流转换 + _AgnoAgentWrapper |
| `engine/hermes.py` | Hermes 引擎：OpenAI SDK + system prompt 注入技能目录 |
| `server/api.py` | FastAPI app：SSE + WS + 直调 + 推送端点 |
| `server/auth.py` | Token 提取（header/cookie/query）→ 调 User API → 构建 AgentContext |
| `server/push.py` | PushManager：token → WebSocket 映射，脚本 POST 推送到前端 |
| `infra/script_executor.py` | subprocess 执行 run.py + 参数拼接 + 结果解析（仅直调用） |
| `mem/factory.py` | MemoryFactory：统一构建 ConversationMemory + MemoryManager |
| `mem/core/conversation_memory.py` | 对话记忆：消息缓冲 + Prefix Checkpoint + 上下文构建 |
| `mem/core/memory_manager.py` | 长期记忆：store/search/forget + HNSW + BM25 + RRF |
| `mem/core/task_list.py` | 任务记忆：两层表 CRUD + 完成判定 + gate 检查 |
| `mem/core/memory_tools.py` | Agent 工具函数：memory_store/search/forget/checkpoint |

## Skill 是什么

一个 Skill 就是一个目录：

```
skills/
└── order-analytics/
    ├── SKILL.md          # 描述文件（LLM 读这个决定是否调用）
    ├── scripts/
    │   └── run.py        # 入口脚本（typer CLI）
    ├── assets/
    │   └── config.yaml   # 配置文件
    └── references/       # 扩展文档（可选）
```

LLM 读 SKILL.md → 决定调用 → subprocess 执行 run.py → 结果返回 LLM

## 两条调用路径

| 路径 | 触发 | 经过 LLM？ | 用途 |
|------|------|:---:|------|
| **Agent 路径** | 用户发消息 | ✅ | 需要理解和决策的复杂任务 |
| **直调路径** | HTTP POST / WS call | ❌ | 确定性操作（查状态、获取建议）|

## 三个 ADR 核心决策

| ADR | 决策 | 一句话 |
|-----|------|--------|
| **0005** | 任务记忆 | 多步 SKILL 的参数/进度从会话上下文抽离到独立表，惰性 gate 防提前执行 |
| **0006** | 自研引擎 | 摆脱 agno/hermes 依赖，用 openai SDK 直连 while 循环 |
| **0008** | 涌现式 SKILL | 拒绝 ACL 权限表，用图可达性做权限，规模越大越划算 |

## 当前状态与演进方向

| 维度 | 现状 | 演进方向 |
|------|------|---------|
| 引擎 | Agno（默认）+ Hermes | 自研 while 循环（ADR-0006）|
| 数据库 | PostgreSQL（ParadeDB）| 已完成从 SurrealDB 迁移 |
| 记忆 | 三层（对话/长期/会话）| + 任务记忆（ADR-0005）|
| 权限 | 仅 user_id 隔离 | 涌现图（ADR-0008）|
| 技能加载 | 全量注入 prompt | 按需检索 Top-N |
| 技能定义 | 人工规划 | 涌现式（使用中自己长出来）|
