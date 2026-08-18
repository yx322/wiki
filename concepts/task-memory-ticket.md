---
title: 任务记忆（Task Memory / Ticket）
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [agent, memory, skillforge, multi-step-skill, task-isolation]
sources: [raw/articles/adr-0005-task-memory.md]
confidence: high
---

# 任务记忆（Task Memory / Ticket）

多步 SKILL 执行过程中的**任务态隔离**机制——把"执行到哪了、收集了哪些参数"从 LLM 的上下文窗口搬到数据库里，LLM 忘了也没关系，DB 帮你记着。

## 问题：多步 SKILL 的任务态没有独立载体

[[skillforge]] 现有记忆系统有对话记忆（session 级）和长期记忆（user 级），但缺少**任务级状态**。

| 痛点 | 场景 | 后果 |
|------|------|------|
| **多任务交叉污染** | 同一会话内并发"数据迁移"和"服务器重启" | 参数混在一起，LLM "拿错剧本" |
| **已收集参数无 buffer** | 多步任务等用户补充信息 | 参数靠 LLM 重新推断，丢信息 |
| **运行前置条件无 gate** | LLM 参数没集齐就调 `run.py` | 脚本执行失败 |

## 设计决策

### 隔离键扩展

```
以前：user_id + session_id     → 同一会话内一切共享
现在：user_id + session_id + task_id → 同一会话内任务各自独立
```

### 关键原则

- **纯数据，非生命周期**：不存在"挂起/续传"，"续传"只是下一次 turn 重新 `memento_read` 读回
- **惰性检查**：只有执行 SKILL 脚本时触发校验，不执行就没检查
- **默认多任务**：同一 session 可并存多个任务，惰性使其零额外成本

### 存储：两层表

```
tasks（薄表，任务级元数据）
├── id（自增短数字 id）
├── user_id, session_id, skill_name
├── status: running | waiting_user | done | aborted
└── created_at, updated_at

task_items（核心表，每一条步骤记录）
├── id（自增）
├── task_id
├── content（步骤描述）
├── attachments（收集到的键值对参数）
├── created_at
└── completed_at（未完成则 NONE）
```

**为什么两层表？** `tasks` 承载任务级元数据，`task_items` 承载步骤列表。单表会被迫把任务级信息冗余进每条 item。

**完成判定归 DB**：

```sql
SELECT VALUE count() FROM task_items
WHERE task_id = $task_id AND completed_at IS NONE;
```

计数为 0 → 任务自动落 `done`。不靠 LLM 判断是否完成。

### 三个注入工具

| 工具 | 作用 |
|------|------|
| `memento_create(task_id?, skill_name, content)` | 无 task_id → 新建任务+首条 item；有 → 追加 item |
| `memento_read` | 整理本会话全部任务为 markdown TODO 列表 |
| `memento_update_status(task_id, item_id, done)` | 改 item 完成状态；无未完成 item → 任务自动 done |

### Gate 机制（惰性硬检查）

```
LLM 调用 memento_create → 收集参数 → 无检查
LLM 调用 memento_read   → 查看进度 → 无检查
LLM 调用 execute_script  → ⚡ 触发 gate
    → 检查该 task 下是否有未完成 item
    → 有 → 返回错误 + 当前任务清单
    → 无 → 放行执行
```

校验失败时返回错误 + 任务清单，LLM 读到即停。不切断 agent 状态机，符合现有 `{data, error, code}` 协议。

## 典型执行流程

```
用户: "帮我部署生产环境"
LLM: memento_create(skill_name="deploy", content="收集服务器地址")
LLM: "请提供服务器地址？"
用户: "10.6.6.88"
LLM: memento_update_status(task_id=1, item_id=1, done=true)
LLM: memento_create(task_id=1, content="收集部署密码")
用户: "先帮我查一下昨天的日志"     ← 中途换话题
LLM: （处理日志查询，任务态安静躺在 DB）
用户: "密码是 abc123"              ← 用户回来
LLM: memento_update_status(task_id=1, item_id=2, done=true)
LLM: execute_script("deploy")      ← gate: 所有 item done ✅ → 放行
```

## 与 Qwen tool_choice 强锁的对比

Qwen 用 `tool_choice` 把 LLM 钉死在"调用某个工具"上。

| 维度 | Qwen tool_choice 强锁 | 任务记忆方案 |
|------|---------------------|------------|
| 路径死锁 | 剥夺其他工具权利，陷入死循环 | 不强锁，LLM 仍可自由调其他工具 |
| 多任务污染 | 只锁工具名，无任务维度 | task_id 隔离，杜绝跨任务污染 |
| 逃生通道 | 无"放弃求助"机制，烧干 token | LLM 可自由放弃，无强制钳死 |
| 适用场景 | 单步/单任务/槽位填充 | 多步/多任务/可中断 |

**核心取舍**：结构性修复（任务态隔离 + DB 硬检查）优于强制约束（tool_choice 强锁）。

## 局限性：闲聊带跑问题

任务记忆解决的是**数据层**问题（参数不丢、任务隔离、完成判定）。

**不能解决的**：LLM 被闲聊带跑后，不会主动想起来去调 `memento_read`——任务态在 DB 里，但没人提醒它。

```
用户: "帮我部署生产环境"
LLM: 开始收集参数...
用户: "今天天气真好"          ← 闲聊
LLM: "是啊天气不错..."       ← 被带跑，忘了部署任务
```

### 完整方案：三层防线

```
第一层：任务记忆       = 可靠的持久存储（数据不丢）
第二层：Agent 状态列   = 可靠的注意力引导（不会忘）→ 见 [[agent-status-bar]]
第三层：gate 硬检查    = 可靠的最终防线（没集齐就跑不了）
```

| | 任务记忆 | + Agent 状态列 |
|--|-------|-------|
| 任务态存在哪 | DB | DB + 每轮注入上下文 |
| 需要 LLM 主动查 | 是 | 不需要（自动注入）|
| 闲聊带跑后能回来 | 需要用户主动提起 | LLM 自己就能看到 |

## 命名约定

| 旧名 | 新名 | 原因 |
|------|------|------|
| WorkingMemory | **ConversationMemory**（对话记忆）| 它只管对话历史 |
| 本方案 | **TaskMemory**（任务记忆）| 认知意义上的"工作记忆" |

## 参见
- [[agent-memory]] — Agent 记忆两层架构
- [[agent-status-bar]] — Agent 状态列（注意力引导层）
- [[skillforge]] — 运行时框架
- [[agent-compound-interest]] — Agent 复利（持久化的价值）

^[raw/articles/adr-0005-task-memory.md]
