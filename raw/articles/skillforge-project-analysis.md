# SkillForge 项目全面解析

## 一、项目概述

**SkillForge** 是一个 **AI Agent 技能运行时环境**（版本 0.1.1）。它的核心理念是：

> 让 AI Agent 拥有可插拔的"技能"（Skills），通过统一的框架加载技能、管理上下文、提供 API 服务，并支持记忆持久化。

简单类比：如果把 AI Agent 比作一个人，那 **SkillForge 就是这个人的"操作系统"**——它管理 Agent 的配置、技能加载、用户认证、对话记忆、API 服务等一切基础设施。

---

## 二、整体架构图

```
用户请求 (HTTP/WebSocket/CLI)
         │
         ▼
┌─────────────────────────────────────────────┐
│                  cli.py                      │  ← 命令行入口
│           (ask / chat / serve / replay)      │
└──────────────┬──────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│              api_server.py                   │  ← HTTP/WS 服务入口
│     (FastAPI: /v1/chat/completions, /ws/chat)│
└──────────────┬──────────────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌──────────────┐
│config.py│ │core.py │ │auth_context.py│   ← 配置 / Agent 创建 / 认证
└────────┘ └───┬────┘ └──────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌──────────────┐
│context │ │context_│ │   replay.py  │   ← 上下文 / 日志收集 / 回放调试
│  .py   │ │collector│ │              │
└────────┘ └────────┘ └──────────────┘
               │
               ▼
┌─────────────────────────────────────────────┐
│              scaffold.py                     │  ← 项目脚手架
│         (templates/project/...)              │
└─────────────────────────────────────────────┘
```

---

## 三、逐文件详解

### 1. `__init__.py` — 包入口，公开 API

```python
__version__ = "0.1.1"
```

这个文件定义了包的版本号，并导出了核心公共 API：

| 导出项 | 来源 | 用途 |
|--------|------|------|
| `AgentContext` | `context.py` | Agent 运行上下文（用户 ID、会话 ID、元数据） |
| `create_agent` | `core.py` | 创建 Agent 实例的核心工厂函数 |
| `create_local_skills` | `core.py` | 从本地目录加载技能 |
| `set_agent_context_env` | `core.py` | 将上下文注入到环境变量 |
| `Settings` / `settings` / `reload_settings` | `config.py` | 全局配置管理 |

**设计意图**：外部使用者只需 `from skillforge import create_agent, settings` 即可，不需要关心内部模块结构。

---

### 2. `__main__.py` — 模块执行入口

```python
from skillforge.cli import app
app()
```

只有 3 行代码，作用是让包可以通过 `python -m skillforge` 执行，等价于运行 `skillforge` 命令。这是 Python 的标准约定。

---

### 3. `config.py` — 全局配置中心

这是整个项目的**配置基石**，使用 `pydantic-settings` 实现，支持三种配置来源（优先级从高到低）：

1. **环境变量**（前缀 `SKILLFORGE__`，嵌套用 `__` 分隔）
2. **TOML 配置文件**（默认 `config.toml`，可通过 `SKILLFORGE_CONFIG` 环境变量覆盖）
3. **默认值**

#### 配置结构一览：

```
Settings（全局配置）
├── agent: AgentConfig        # Agent 名称、描述、指令、调试模式
├── skills: SkillsConfig      # 技能加载方式 (local/remote/both)、路径
├── surreal: SurrealDBConfig  # SurrealDB 数据库连接（记忆存储）
├── server: ServerConfig      # 服务端口、CoT/工具调用输出格式、WS ping
├── llm: LLMConfig            # 模型名 (如 openai:gpt-4o-mini)、API Key、Base URL
├── logging: LoggingConfig    # 日志级别、格式、上下文日志文件
├── cli: CliConfig            # CLI 模式的用户 ID、会话 ID、Cookie/Header
└── auth: AuthConfig          # 认证方式：Header/Cookie/Query 参数名
```

#### 关键设计细节：

- **LLM 模型解析**：`model = "openai:gpt-4o-mini"` 格式，冒号前是 provider，后面是模型 ID
- **SurrealDB**：用于 Agent 的**记忆持久化**，存储对话历史和会话信息
- **ServerConfig** 中的 `tool_call_format` 使用 Jinja2 模板语法，可自定义工具调用的输出格式
- **AuthConfig** 的 `auth_method` 是一个列表，按顺序尝试认证方式

---

### 4. `context.py` — Agent 运行上下文

`AgentContext` 是一个 Pydantic BaseModel，用于在请求生命周期内传递上下文信息：

```python
class AgentContext(BaseModel):
    user_id: Optional[str] = None       # 用户标识
    session_id: Optional[str] = None    # 会话标识
    metadata: Dict[str, Any] = {}       # 扩展元数据（cookies、token 等）
```

**核心方法**：
- `set_cookie(key, value)` / `get_cookie(key)` — 管理 cookies（存在 `metadata["cookies"]` 中）
- `set_metadata(key, value)` / `get_metadata(key)` — 管理任意元数据
- `to_dict()` — 序列化为字典

**设计意图**：使用 `metadata` 字典而不是硬编码字段，这样新增信息不需要修改类定义。比如技能需要传递 `access_token`、`transport` 等信息，都可以通过 `set_metadata` 动态添加。

---

### 5. `auth_context.py` — 认证与上下文构建

这个模块负责**从 HTTP 请求中提取认证信息**并构建 `AgentContext`。

#### 核心函数：

| 函数 | 作用 |
|------|------|
| `extract_token(headers, cookies, query_params)` | 按 `auth_method` 配置的顺序，从 Header/Cookie/Query 参数中提取 Token |
| `build_context_with_token(...)` | 构建包含 Token 的 AgentContext，Token 存入 `metadata["access_token"]` |
| `build_context_from_request(request)` | 从 Starlette Request/WebSocket 自动提取 headers/cookies/query_params 并构建上下文 |
| `build_context_from_cli_config(...)` | CLI 模式下从环境变量读取 Cookie/Header 值构建上下文 |

**认证流程**：
```
请求到达 → extract_token() 按 [query, header, cookie] 顺序查找
         → 找到 Token → 存入 context.metadata["access_token"]
         → 技能执行时通过环境变量 CONTEXT_METADATA_ACCESS_TOKEN 获取
```

**关键设计**：运行时只负责**透传** Token，不负责验证。验证和获取用户信息是具体 Skill 的职责。

---

### 6. `core.py` — Agent 创建核心

这是整个项目最核心的模块，负责**创建 Agent 实例**和**管理上下文环境变量**。

#### 关键组件：

**① 上下文环境变量注入（解决并发问题）**

```python
_current_context: ContextVar[Optional[AgentContext]] = ContextVar("current_context", default=None)
```

使用 Python 的 `ContextVar` 来跟踪每个请求的上下文，避免服务模式下多个并发请求互相干扰。

`flatten_context_env()` 将上下文展平为环境变量字典：
- `user_id` → `CONTEXT_USER_ID`
- `metadata.cookies.token` → `CONTEXT_METADATA_COOKIES_TOKEN`
- 同时保留 `CONTEXT_JSON` 供需要完整数据的场景

**② subprocess.run 猴子补丁**

```python
_original_subprocess_run = subprocess.run

def _context_aware_subprocess_run(*args, **kwargs):
    ctx_env = get_context_env_for_subprocess()
    if ctx_env:
        env = kwargs.get('env', os.environ.copy())
        env.update(ctx_env)
        kwargs['env'] = env
    return _original_subprocess_run(*args, **kwargs)

subprocess.run = _context_aware_subprocess_run
```

**为什么需要这个？** 因为技能（Skills）可能通过子进程执行（CLI 技能），需要将当前请求的上下文注入到子进程的环境变量中。直接修改全局 `os.environ` 在并发场景下会互相覆盖，所以通过猴子补丁 `subprocess.run`，在每次调用子进程时自动注入当前请求的上下文环境变量。

**③ `create_agent()` — Agent 工厂函数**

这是最核心的函数，创建一个完整的 Agent 实例：

```python
def create_agent(llm, agent_config, skills, debug_mode, reasoning, 
                 stream, context, memory_storage, ...) -> Agent:
```

流程：
1. **解析 LLM 配置**：`_resolve_model()` 将 `"openai:gpt-4o-mini"` 解析为 provider + model_id + api_key + base_url
2. **创建模型实例**：`OpenAIChat(id=..., api_key=..., base_url=...)`
3. **解析 reasoning 配置**：`_resolve_reasoning()` — 阿里云 qwen 模型默认禁用 reasoning
4. **配置 Agent 参数**：名称、描述、指令、技能、上下文、记忆存储
5. **记忆功能**：如果提供了 `user_id` 和 `session_id`，启用 SurrealDB 记忆存储和历史消息

**④ `create_local_skills()` — 本地技能加载**

```python
def create_local_skills(paths: list[str] | str = ".") -> Skills:
    return Skills(loaders=[LocalSkills(path=p) for p in paths])
```

使用 agno 框架的 `LocalSkills` 从指定目录加载技能。每个技能目录需要包含 `SKILL.md` 文件。

---

### 7. `api_server.py` — API 服务器

这是项目的**对外服务层**，提供 OpenAI 兼容的 API 接口。

#### API 端点：

| 端点 | 方法 | 用途 |
|------|------|------|
| `/v1/chat/completions` | POST | OpenAI 兼容的流式聊天接口（SSE） |
| `/ws/chat` | WebSocket | WebSocket 聊天接口 |
| `/v1/models` | GET | 列出可用模型 |
| `/health` | GET | 健康检查 |

#### 请求/响应模型（OpenAI 兼容）：

```python
class ChatCompletionRequest(BaseModel):
    model: str = "agent"
    messages: List[ChatMessage]
    stream: Optional[bool] = False
    ...

class ChatCompletionResponse(BaseModel):
    id: str
    object: str = "chat.completion"
    choices: List[ChatChoice]
    usage: Optional[ChatCompletionUsage] = None
```

#### SSE 流式输出流程：

```
客户端请求 → 创建 AgentContext → 创建 Agent 实例
         → agent.run(stream=True, stream_events=True)
         → 遍历事件流：
            ├── ReasoningContentDeltaEvent → 思考过程（CoT）
            ├── ToolCallStartedEvent → 工具调用开始
            ├── ToolCallCompletedEvent → 工具调用完成
            ├── RunContentEvent → 普通内容输出
            └── RunCompletedEvent → 运行完成（含 token 用量）
         → SSEFormatter 格式化输出
         → StreamingResponse 返回给客户端
```

#### 格式化器设计：

`SSEFormatter` 和 `WebSocketFormatter` 是两个静态类，负责将 Agent 事件格式化为不同传输协议的格式。`_generate_agent_events()` 是共享的生成器，通过传入不同的格式化器来输出 SSE 或 WebSocket 格式，实现了**逻辑复用**。

#### 每请求创建 Agent：

```python
request_agent = create_agent_for_request(context=context)
```

**不是全局共享一个 Agent**，而是每个请求创建独立的 Agent 实例，这样可以：
- 支持多用户隔离
- 每个用户有独立的记忆存储
- 避免并发请求互相干扰

---

### 8. `context_collector.py` — 上下文日志收集器

`ContextCollector` 在一次 LLM 请求过程中收集完整上下文，并写入 JSONL 日志文件。

#### 收集的数据：

```python
{
    "response_id": "chatcmpl-xxx",
    "model": "gpt-4o-mini",
    "input_messages": [...],        # 用户输入的消息
    "llm_messages": [...],          # 实际发送给 LLM 的完整消息（含 system prompt、工具调用结果）
    "output_content": "...",        # Agent 输出内容
    "reasoning": "...",             # 思考过程
    "tool_calls": [...],            # 工具调用记录
    "agent_instructions": "...",    # Agent 系统提示词
    "status": "completed",          # 请求状态
    "elapsed_ms": 1234,             # 耗时（毫秒）
    "usage": {...},                 # Token 用量
}
```

**用途**：这些日志是 `replay.py` 回放调试的数据来源。

---

### 9. `replay.py` — 请求回放调试

从 JSONL 上下文日志中还原 LLM 请求，直接调用 OpenAI 兼容 API 进行调试。

#### 核心功能：

| 命令 | 用途 |
|------|------|
| `skillforge replay logs/context.jsonl` | 回放最后一条记录 |
| `skillforge replay logs/context.jsonl --index 0` | 回放第一条 |
| `skillforge replay logs/context.jsonl --dry-run` | 只打印 curl 命令，不执行 |
| `skillforge replay logs/context.jsonl --list` | 列出所有条目 |
| `skillforge replay-auto` | 自动从 config.toml 读取配置回放 |

#### 消息还原逻辑（`_reconstruct_messages`）：

1. 优先使用 `llm_messages`（完整还原实际发送给 LLM 的消息）
2. 如果没有，则从 `agent_instructions` + `input_messages` + `tool_calls` 重建

**这是调试利器**：当 Agent 行为异常时，可以通过回放日志还原完整的请求上下文，定位问题。

---

### 10. `scaffold.py` — 项目脚手架

`create_project()` 函数从 `templates/project/` 目录复制模板文件，创建新的 SkillForge 项目：

```python
def create_project(target_dir, project_name, force=False) -> Path:
    # 复制模板文件，替换 {PROJECT_NAME} 占位符
    for src_path in TEMPLATES_DIR.rglob("*"):
        content = src_path.read_text(encoding="utf-8")
        content = content.replace("{PROJECT_NAME}", project_name)
        dest_path.write_text(content, encoding="utf-8")
```

通过 `skillforge init --name my-agent` 命令调用。

---

### 11. `cli.py` — 命令行接口

使用 `typer` 框架提供 CLI 命令：

| 命令 | 用途 |
|------|------|
| `skillforge ask <dir> "问题"` | 单次提问 |
| `skillforge chat` | 交互式对话 |
| `skillforge serve <dir>` | 启动 API 服务 |
| `skillforge init` | 初始化新项目 |
| `skillforge replay <log>` | 回放调试 |
| `skillforge replay-auto` | 自动回放 |
| `skillforge version` | 显示版本 |
| `skillforge info` | 显示当前配置 |

---

### 12. 模板文件

| 文件 | 用途 |
|------|------|
| `config.toml` | 项目配置模板，包含所有配置项的默认值和注释 |
| `Dockerfile` | 标准 Docker 构建文件 |
| `Dockerfile.nu` | 基于 buildah + nushell 的构建脚本（更灵活） |
| `x.nu` | Nushell 辅助脚本，提供技能同步、测试、镜像构建等快捷命令 |
| `.gitignore` | Git 忽略配置 |
| `README.md` | 项目说明模板 |

---

## 四、核心数据流

### 场景 1：用户通过 API 发起聊天

```
1. 客户端 POST /v1/chat/completions
2. api_server.py: chat_completions()
   ├── build_context_from_request() → 从请求中提取 Token/Cookie → AgentContext
   ├── context.set_metadata("transport", "sse")
   └── create_agent_for_request(context) → 创建 Agent
       ├── create_skills() → 加载本地技能
       ├── create_agent() → 配置 LLM、技能、记忆存储
       └── set_agent_context_env(context) → 设置 ContextVar
3. _generate_agent_events() → 流式处理 Agent 事件
   ├── ContextCollector 收集完整上下文
   ├── agent.run(stream=True) → 执行 Agent
   ├── 遍历事件流 → SSEFormatter 格式化 → yield 给客户端
   └── collector.write() → 写入 JSONL 日志
4. StreamingResponse 返回 SSE 流
```

### 场景 2：技能执行时获取上下文

```
1. Agent 调用某个 CLI 技能（通过 subprocess.run）
2. _context_aware_subprocess_run() 拦截调用
3. get_context_env_for_subprocess() → 从 ContextVar 获取当前请求的上下文
4. flatten_context_env() → 展平为环境变量字典
5. 注入到 subprocess.run 的 env 参数
6. 技能子进程通过 os.environ["CONTEXT_METADATA_ACCESS_TOKEN"] 等获取上下文
```

---

## 五、关键设计决策总结

| 设计决策 | 原因 |
|----------|------|
| **每请求创建 Agent** | 多用户隔离、独立记忆、避免并发冲突 |
| **ContextVar + subprocess 猴子补丁** | 服务模式下并发请求的上下文隔离 |
| **metadata 字典而非硬编码字段** | 灵活扩展，新增信息无需修改类定义 |
| **OpenAI 兼容 API** | 可直接接入任何支持 OpenAI API 的客户端 |
| **SSE + WebSocket 双协议** | SSE 适合 Web 前端，WebSocket 适合移动端/实时场景 |
| **JSONL 上下文日志 + 回放** | 生产环境调试利器，可还原完整请求上下文 |
| **TOML 配置 + 环境变量** | 配置灵活，环境变量优先级最高便于部署 |
| **Token 透传不验证** | 职责分离，运行时只负责传递，验证由具体 Skill 实现 |

---

## 六、依赖关系

项目依赖以下核心外部库：

| 库 | 用途 |
|----|------|
| `agno` | AI Agent 框架，提供 Agent、Skills、LocalSkills 等核心抽象 |
| `fastapi` + `uvicorn` | HTTP/WS 服务 |
| `pydantic-settings` | 配置管理（支持 TOML + 环境变量） |
| `typer` | CLI 框架 |
| `jinja2` | 模板引擎（工具调用输出格式化） |
| `shortuuid` | 生成短响应 ID |
| `mem` (内部库) | SurrealDB 记忆存储适配 |

---

这就是 SkillForge 项目的完整解析。如果对某个模块还有疑问，或者想深入了解某个具体功能的实现细节，随时告诉我！
