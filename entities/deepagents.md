---
title: DeepAgents（LangChain deepagents 库）
created: 2026-09-07
updated: 2026-09-07
type: entity
tags: [deepagents, langchain, langgraph, agent, skill]
sources: [raw/articles/deepagent-langchain-dialog.md]
confidence: high
contested: false
---

# DeepAgents

LangChain 团队 2025 年年中开源的深度任务 Agent 库（`pip install deepagents`），本质是 **LangGraph 的一层薄封装**——不发明新引擎，用 prompt engineering + 一组内置工具把"长任务深度执行"固化为开箱即用的东西。

## 定位：模型编排框架，不是推理引擎

| 层 | 组件 | 职责 |
|---|---|---|
| 编排层 | deepagents | agent 模式的 prompt + 工具封装（规划工具、子代理、虚拟文件系统） |
| 编排层 | LangGraph | 图执行引擎、状态管理、持久化、HITL |
| 抽象层 | langchain-core | 统一 LLM/工具/消息接口 |
| 推理层 | Claude/GPT API、vLLM、Ollama | 真正执行模型（矩阵运算、生成 token） |

**关键区分**：编排框架"不跑模型，只组织模型调用"。模型调用次数 ≠ 任务数——一个深度任务可能触发几十上百次 API 调用。

## 与 LangGraph 的对应关系

| deepagents 能力 | LangGraph 底层机制 |
|---|---|
| Agent 主循环 | `langgraph.prebuilt.create_react_agent`（ReAct 架构） |
| 状态（messages + files + todos） | 自定义 State 继承 MessagesState，加 `files: dict` 和 `todos: list` |
| Subagents（task 工具） | 各自编译好的 subgraph，通过工具调用挂进主图 |
| HITL | `interrupt()` + `Command(resume=...)` |
| 会话持久化 | checkpointer（同一线程 ID 跨会话恢复） |
| 虚拟文件系统 | 状态里的 files 字段 + 内置 ls/read/write/edit 工具 |

## 真实核心能力（官方）

- **Planning tool**（内置 `write_todos` 待办清单）
- **Subagents**（`task` 工具派生子代理）
- **虚拟文件系统**（`ls / read_file / write_file / edit_file`）
- **上下文摘要压缩**（防上下文膨胀）
- **HITL + 后端持久化**（LangGraph checkpoint）

设计精髓：**用几个简单原语（todo list + 子代理 + 文件系统）+ 超长上下文 LLM 实现"深"效果，而不是堆复杂机制**。它的 virtual filesystem 本质上是一个内置的 proto-skill 机制（模型按需 read/write 文件）。

## ⚠️ 知识纠偏（对话中的诚实澄清）

第一轮对话里的这些 API 是**示意性虚构**，官方不存在：
- `create_deep_agent(planning_strategy="hierarchical", ...)` 参数
- `SemanticCache / ToolCache / ReasoningCache`（官方只有精确匹配的 LLM cache，即 `set_llm_cache`）
- `HierarchicalMemory` 三层记忆

但它们是合理的工程思路，实现要点见 [[agent-cache-memory-optimization]]。

## 参见

- [[skill-vs-framework]] — Skill 模式 vs 框架路线：两种哲学的对比（本对话核心结论）
- [[agent-cache-memory-optimization]] — 语义缓存/工具缓存/推理链缓存/分层记忆的实现思路
- [[skillforge]] — 用户的 Skill 化 Agent 运行时
- [[emergent-skill]] — 涌现式技能

^[raw/articles/deepagent-langchain-dialog.md]
