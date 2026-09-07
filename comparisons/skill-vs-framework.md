---
title: Skill 模式 vs 框架路线
created: 2026-09-07
updated: 2026-09-07
type: comparison
tags: [skill, langgraph, deepagents, agent, orchestration, philosophy]
sources: [raw/articles/deepagent-langchain-dialog.md]
confidence: high
contested: false
---

# Skill 模式 vs 框架路线

Agent 设计的两条路线，根本分歧只有一个问题：**Agent 的流程智能放在哪里？**

```
编排智能的位置：
外部框架（代码里）◄──────────────────────► 模型内部（上下文里）

  LangGraph 传统用法    deepagents（中间地带）    Claude Code + Agent Skills
  状态机写死在代码里      少量内置prompt模式        流程智能全在模型
  每个节点显式定义       +图引擎兜底               +SKILL.md 里
```

## Skill 模式的本质

一个 Skill 是自包含的能力包（SKILL.md + scripts/ + references/），核心机制是**渐进式披露**：

```
第0层: 系统提示里只放每个 skill 的 name+description
       ≈ 几十个 token/skill，模型只知道"我有什么能力"
              ▼  模型判断：这个任务需要 pdf-report
第1层: 加载该 SKILL.md 的完整指令（几百~几千 token）
              ▼  指令说"需要先跑 generate.py --prepare"
第2层: 按需读取脚本/参考资料，甚至读代码学习怎么调用
```

**模型自己决定什么时候加载什么**——编排逻辑不在框架代码里，而在模型 + SKILL.md 里。

## 两种哲学对照

| 维度 | 框架路线（LangGraph/deepagents） | Skill 路线 |
|---|---|---|
| 世界观 | Agent 是软件系统，要工程化管控 | Agent 是能力很强的模型，给它说明书就行 |
| 确定性来源 | 代码结构（图、节点、边） | 模型的判断力 + 写得好的指令 |
| 复杂度放在 | 代码里 | 文档里 |
| 失败时怪谁 | 框架没编排好 | prompt/skill 写得不好 |
| 换强模型后 | 架构显得臃肿 | 表现自动变好 |
| 换弱模型后 | 架构照样兜底 | 崩溃式退化 |

## 职责重分配

| 职责 | 框架路线 | Skill 路线 |
|---|---|---|
| 任务规划 | LangGraph 图结构 / todo 工具 | 模型读 SKILL.md 指令自己规划 |
| 工具调用决策 | ReAct 循环判断 | 模型直接决定（同 function calling） |
| 记忆 | 框架的 Memory / checkpointer | 渐进披露 + 会话历史 |
| 领域知识 | 散落在代码和 prompt 拼接里 | 集中封装在 SKILL.md（核心优势） |
| 能力扩展 | 改框架代码、加图节点 | 丢一个新文件夹进去 |

## Skill 路线的真实权衡

**优势**：
- 能力扩展零代码：新技能 = 新文件夹
- 上下文经济：用哪本翻哪本，不全塞 system prompt
- 知识可版本化：SKILL.md 纯文本，git 管理、可 review
- 框架锁定极轻：核心资产不绑框架，可带走

**代价**（恰是 LangGraph 的强项）：
- 持久化/断点恢复要自己设计状态落盘
- HITL 人工介入要自己实现"暂停等审批"
- skill 选取靠模型判断，选错整条路走偏（弱模型故障率明显上升）
- 多步执行黑盒倾向，要自己埋追踪

## 行业演进：光谱整体右移

- 2023：模型弱，LangChain 用链式代码硬拗智能感
- 2024：LangGraph 承认现实——链太死板，改成"带循环的图"
- 2025：deepagents 砍掉复杂图编排，只留最小骨架，规划交给模型 + todo 工具
- 同年：Anthropic Agent Skills 走到最右端，连骨架都最小化，能力以文档注入

**Skill 路线 = 赌模型会持续变强的位置**，且行业在验证这个赌注。

## 判断标准与退路

```
任务时长 < 几分钟、人在线          → skill 模式完全够 ✅
任务要跑很久、无人值守             → 需要补持久化 ⚠️
强合规/审计要求（每步可追溯、人工介入）→ 需要外部强制结构 ⚠️
```

**退路**：skill 模式核心资产和执行骨架解耦——需要无人值守长任务时，最外面包一层薄 LangGraph（或自写带 checkpoint 的循环），skills 完全不用动。反向（深框架拆出 skill 化）成本高得多。

常见混合形态：LangGraph 薄编排（循环 + checkpointer + HITL）+ Agent 节点内挂 Skills 目录（渐进披露）。

## 参见

- [[deepagents]] — 中间地带的代表实现
- [[skillforge]] — 用户自己的 Skill 化运行时（薄循环 + SKILL.md）
- [[emergent-skill]] — 涌现式技能演进方向
- [[skill-direct-invocation]] — 确定性操作绕过模型
- [[prefix-checkpoint]] — skillforge 的会话压缩方案
- [[task-memory-ticket]] — skillforge 对持久化短板的补法

^[raw/articles/deepagent-langchain-dialog.md]
