---
title: SkillForge
created: 2026-07-21
updated: 2026-12-19
type: entity
tags: [agent, skillforge, architecture, adr-0006]
sources: [raw/articles/skillforge-project-analysis.md]
confidence: high
---

# SkillForge

AI Agent 技能运行时环境（v0.1.1），让 AI Agent 拥有可插拔的"技能"。

## 核心定位

如果把 AI Agent 比作一个人，SkillForge 就是这个人的"操作系统"——管理配置、技能加载、用户认证、对话记忆、API 服务等一切基础设施。

## 架构演进

### 当前架构（ADR-0006 完成后）

```
用户请求 (HTTP/WebSocket/CLI)
         ↓
     api_server.py (FastAPI)
         ↓
  config.py / core.py / auth_context.py
         ↓
  context.py (Agent 上下文管理)
         ↓
  engine/ (自研引擎，ADR-0006)
    ├─ llm.py (while 工具循环 + openai SDK 直连)
    └─ skill.py (技能加载/解析/gate 机制)
         ↓
  mem/ (记忆系统，PostgreSQL)
    ├─ conversation.py (对话记忆 ConversationMemory)
    ├─ task.py (任务记忆 tasks/task_items)
    └─ long_term.py (长期记忆 memories)
```

### 架构演进历程

| 阶段 | 引擎 | 记忆存储 | 关键决策 |
|------|------|----------|----------|
| 初始版本 | agno + hermes 双引擎 | SurrealDB | 快速原型 |
| ADR-0005 | agno + hermes | SurrealDB | 任务记忆设计 |
| ADR-0006 | **自研引擎** | SurrealDB | 摆脱 agno/hermes 依赖 |
| ADR-0007 | 自研引擎 | **PostgreSQL** | 记忆存储迁移 |

## 关键设计决策

| 决策 | 原因 |
|------|------|
| **自研 tool-call 循环**（ADR-0006）| agno 只剩循环价值，依赖不值得 |
| **PostgreSQL 替代 SurrealDB**（ADR-0007）| 生产稳定性、生态成熟 |
| 每请求创建 Agent | 多用户隔离、独立记忆、避免并发冲突 |
| ContextVar + subprocess 猴子补丁 | 并发请求的上下文隔离 |
| metadata 字典而非硬编码字段 | 灵活扩展 |
| OpenAI 兼容 API | 可接入任何支持 OpenAI API 的客户端 |
| SSE + WebSocket 双协议 | SSE 适合 Web，WebSocket 适合移动端 |

## 自研引擎（ADR-0006）

### 核心实现

```python
# engine/llm.py 核心循环
messages = [system, *working_memory_context, user]
while True:
    resp = client.chat.completions.create(model, messages, tools=tool_defs, stream=True)
    # 累积流式增量，emit 事件
    if not resp.choices[0].message.tool_calls:
        break  # 无工具调用，结束
    for tc in tool_calls:
        result = execute(tc)  # 执行工具
    messages.append(assistant)
    messages.append({role: 'tool', content: result})
    # 继续循环
```

### 三个内置 Skill 工具

1. **`get_skill_instructions(skill_name)`** — 加载技能完整指令
2. **`get_skill_reference(skill_name, reference_path)`** — 访问参考文档
3. **`run_skill_script(skill_name, script_path, args)`** — 执行脚本（带任务记忆 gate）

### 关键特性

- **ConversationMemory 注入位**：循环前注入 checkpoint + 增量消息
- **记忆事件拦截**：RunCompleted 时写 assistant 消息到数据库
- **任务记忆 gate**：`run_skill_script` 执行前检查任务完成状态
- **最大步数限制**：`max_steps=10` 防死循环

## 记忆系统

使用 **PostgreSQL** 存储三层记忆（从 SurrealDB 迁移完成，ADR-0007）：

| 记忆层 | 表 | 职责 |
|--------|-----|------|
| 对话记忆 | `session_messages` + `session_checkpoints` | Prefix Checkpoint 压缩 |
| 任务记忆 | `tasks` + `task_items` | 多步任务状态隔离（ADR-0005）|
| 长期记忆 | `memories` | 跨会话知识积累 |

## CLI 命令

| 命令 | 用途 |
|------|------|
| `skillforge ask <dir> "问题"` | 单次提问 |
| `skillforge chat` | 交互式对话 |
| `skillforge serve <dir>` | 启动 API 服务 |
| `skillforge init` | 初始化新项目 |

## 参见
- [[agent-memory]] — 记忆架构
- [[task-memory-ticket]] — 任务记忆（ADR-0005）
- [[prefix-checkpoint]] — Prefix Checkpoint 压缩机制
- [[agent-compound-interest]] — Agent 复利

^[raw/articles/skillforge-project-analysis.md]
