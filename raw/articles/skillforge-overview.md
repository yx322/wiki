# SkillForge 项目总览与架构解析

> 基于 D:\skillforge 项目文档整理
> 日期：2026-08-20

---

## 一、项目定位

**SkillForge 是一个 AI Agent 技能运行时环境**，提供 OpenAI 兼容的 API 服务和命令行工具，支持通过自然语言调用各种业务技能（Skills）。

核心理念：**SKILL 化一切** — 应用能力不再是零散的 API，而是封装好的、带语义描述（SKILL.md）的 CLI 命令行程序。

## 二、技术栈

| 组件 | 选型 | 说明 |
|------|------|------|
| 语言 | Python ≥ 3.10 | |
| Web 框架 | FastAPI + Uvicorn | SSE + WebSocket |
| Agent 框架 | Agno（默认）/ Hermes | 可插拔，正在迁移到自研引擎 |
| 数据库 | PostgreSQL（ParadeDB 镜像）| 记忆 + 会话存储（从 SurrealDB 迁移中）|
| 配置 | pydantic-settings + config.toml | 环境变量 > TOML > 默认值 |
| 构建 | hatchling | 包路径 src/skillforge + src/mem |

## 三、目录结构

```
skillforge/
├── src/skillforge/           # 主包
│   ├── __init__.py           # 包入口
│   ├── cli.py                # CLI（typer）：ask / chat / serve / init
│   ├── config.py             # 所有配置定义（pydantic-settings）
│   ├── context.py            # AgentContext（user_id, session_id, metadata）
│   ├── core.py               # Agent 创建 + subprocess 拦截
│   ├── events.py             # 流式事件基类（引擎无关）
│   ├── engine/               # 引擎层（可插拔）
│   │   ├── agno.py           # Agno 引擎
│   │   └── hermes.py         # Hermes 引擎
│   ├── server/               # API 服务层
│   │   ├── api.py            # FastAPI app（SSE + WS + 直调接口）
│   │   ├── auth.py           # Token 提取 + AgentContext 构建
│   │   └── push.py           # PushManager（WebSocket 推送）
│   └── infra/                # 基础设施工具
│       ├── script_executor.py # subprocess 执行 run.py
│       └── tool_format.py    # Jinja2 工具调用格式渲染
│
└── src/mem/                  # 记忆系统包
    ├── factory.py            # MemoryFactory（统一构建入口）
    ├── storage/postgres.py   # PostgresStorage 基类
    └── core/
        ├── memory_manager.py # 长期记忆 CRUD + 混合检索
        ├── conversation_memory.py # 对话记忆（Prefix Checkpoint）
        ├── task_list.py      # 任务记忆（两层表 + gate）
        └── memory_tools.py   # Agent 工具函数
```

## 四、API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v1/chat/completions` | POST (SSE) | OpenAI 兼容流式聊天 |
| `/ws/chat` | WebSocket | WebSocket 聊天 + 确定性调用 |
| `/skills/{name}` | POST | 直接调用 Skill 脚本 |
| `/skills` | GET | 返回 skills_registry |
| `/push` | POST | 向指定 token 的 WS 客户端推送 |
| `/health` | GET | 健康检查 |

## 五、核心设计哲学

### 5.1 架构铁律：mem 模块厚，agent 适配层薄

```
mem 模块（厚）：封装所有记忆逻辑
  ↓ 暴露干净接口
agent 适配层（薄）：只做三件事
  1. memory.push_user_message(user)
  2. agent.additional_input = memory.get_context()
  3. memory.write_assistant_message(assistant)
```

### 5.2 物理隔离 + 逻辑中控

- FastAPI 作为网关
- Agent 引擎作为大脑
- PostgreSQL 作为记忆与状态存储
- Docker/容器作为执行沙箱

### 5.3 带外身份注入

- **严禁将 USER_ID 作为提示词参数**
- 统一通过环境变量注入派生进程
- 封死提示词注入漏洞

## 六、数据流（请求级）

```
用户请求
  ↓
SSE/WS 路由
  ↓
create_agent_for_request()
  → ConversationMemory (per session)
  → MemoryManager (per user)
  → create_memory_tools(mm) → tools
  → create_agent(session_memory=cm)
    → _AgnoAgentWrapper(agent, wm)
  ↓
_generate_agent_events(agent, messages, ...)
  ↓
Agent 调用工具 / 返回结果
  ↓
流式输出 + 记忆写入
```

## 七、subprocess 环境变量注入

Agno 不支持为 Agent 设置环境变量，框架通过**拦截 subprocess.run / subprocess.Popen** 实现：

- `core.py` 导入时即 patch `subprocess.run` 和 `subprocess.Popen`
- 从 `ContextVar` 获取当前请求上下文 → 展平为 `CONTEXT_*` 环境变量
- 注入到子进程 `env` 中，不影响全局 `os.environ`

## 八、已知限制

1. **subprocess 拦截为临时方案** — 待 Agno 原生支持后迁移
2. **Hermes 无原生技能机制** — 靠 system prompt 注入目录
3. **SurrealDB → PostgreSQL 迁移中** — 运维复杂度和语法不稳定
4. **正在自研引擎** — 摆脱 agno/hermes 依赖（ADR-0006）
