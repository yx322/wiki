# ADR-0005 任务记忆（Task Memory）讨论记录

> 日期：2026-08-18
> 参与者：orbit（用户）、AI
> 基于：ADR-0005 会话级任务票据（ticket）—— 多步 SKILL 的任务态隔离

---

## 一、ADR 核心内容解析

### 1.1 解决什么问题？

skillforge 现有记忆系统有三层：

| 层 | 表 | 隔离键 |
|---|---|---|
| 对话记忆 | `session_messages` / `session_checkpoints` | user_id + session_id |
| 长期记忆 | `memories` | user_id |
| 会话记忆 | `agno_sessions` | session_id |

**缺口**：当一个多步 SKILL 执行到一半（比如"收集了 3 个参数，还差 2 个"），这些**任务进度状态**没有独立载体，被摊在会话上下文里。

### 1.2 三个具体痛点

| 问题 | 场景 | 后果 |
|------|------|------|
| **多任务交叉污染** | 同一会话内同时发起"数据迁移"和"服务器重启" | 两个任务的参数和步骤进度混在一起，LLM "拿错剧本" |
| **已收集参数无 buffer** | 多步任务走到"等用户补充信息" | 参数只能靠 LLM 在后续 turn 重新推断，丢信息 |
| **运行前置条件无 gate** | LLM 在参数没集齐时就调了 `run.py` | 脚本执行失败，无机制阻止 |

### 1.3 决策：新增"任务记忆"

在 mem 模块新增**第三类记忆**——任务记忆（task list / ticket）。

**隔离键扩展**：`user_id + session_id` → `user_id + session_id + task_id`

**关键原则**：
- 任务态是**纯数据，不是生命周期**——不存在"挂起/续传"机制
- **惰性检查**——只有执行 SKILL 脚本时才触发校验，不执行就没检查
- **默认支持多任务**——同一 session 可并存多个任务，零额外成本

### 1.4 存储设计：两层表

```
tasks（薄表，任务级元数据）
├── id（自增，短数字 id）
├── user_id, session_id
├── skill_name
├── status: running | waiting_user | done | aborted
└── created_at, updated_at

task_items（核心表，列表模式每一条记录）
├── id（自增）
├── task_id
├── content（任务步骤描述）
├── attachments（option<object>，收集到的键值对参数）
├── created_at
└── completed_at（未完成则 NONE）
```

**为什么用两层表而不是单表？**
- 单表会被迫把 `skill_name`、`status` 冗余进每条 item，或丢弃任务级信息

**为什么用 SurrealDB 而不是 Redis？**
- skillforge 无 Redis 依赖，为单一功能引入新存储不合算
- 任务态是**必须落盘**的持久数据，SurrealDB 已满足
- 同库 = 同事务域、同隔离模型、同运维面

**为什么用列表模式而不是 markdown 整读整写？**

| 列表模式 | markdown 整读整写 |
|----------|------------------|
| LLM 每次只 create/update 一条 item | 每次整篇重写 |
| 完成判定归 DB（`completed_at IS NONE` 计数）| 靠 LLM 判断是否完成 |
| 短自增 id，操作可靠 | LLM 容易写错长文本 |

**完成判定 SQL**：
```sql
SELECT VALUE count() FROM task_items
WHERE task_id = $task_id AND completed_at IS NONE;
```
计数为 0 → 任务自动落 `done`。

### 1.5 三个注入工具

| 工具 | 作用 |
|------|------|
| `memento_create(task_id?, skill_name, content)` | 无 task_id → 新建任务+首条 item；有 → 追加 item |
| `memento_read` | 整理本会话全部任务为 markdown TODO 列表 |
| `memento_update_status(task_id, item_id, done)` | 改 item 完成状态；无未完成 item → 任务自动 done |

### 1.6 Gate 机制（惰性硬检查）

```
LLM 调用 memento_create → 收集参数 → 无检查
LLM 调用 memento_read   → 查看进度 → 无检查
LLM 调用 execute_script  → ⚡ 触发 gate
    → 检查该 task 下是否有未完成 item
    → 有 → 返回错误 + 当前任务清单
    → 无 → 放行执行
```

**设计哲学**：
- 不执行脚本就无检查，与"脚本由 agent 自主调用"的现状一致
- 校验失败时返回错误 + 任务清单，LLM 读到即停，不需要另调 `memento_read`
- 不切断 agent 状态机，符合现有 `{data, error, code}` 协议

### 1.7 命名变更

| 旧名 | 新名 | 原因 |
|------|------|------|
| WorkingMemory | **ConversationMemory**（对话记忆）| 它只管对话历史 |
| 本 ADR 所述任务态 | **TaskMemory**（任务记忆）| 这才是认知意义上的"工作记忆" |

### 1.8 被否的替代方案：Qwen 的 tool_choice 强锁

Qwen 用 `tool_choice` 把 LLM 钉死在"调用某个工具"上。

**适用边界**（三种场景成立）：
1. 单步/单任务 Agent（无需中途搜数）
2. 极易被闲聊带偏的会话
3. 槽位填充表单任务（唯一任务就是补参数）

**三个致命问题**：

| 死穴 | 原因 |
|------|------|
| **路径死锁** | 多步任务前几步需要调记忆/信息收集工具，强锁后模型被剥夺其他工具权利，陷入死循环 |
| **多任务拿错剧本** | 强锁只锁工具名，没有任务维度，两个并发任务参数污染 |
| **无逃生通道** | 用户不合理时（脚本不存在、不提供密码），模型没"放弃求助"的机制，烧干 token |

**任务记忆方案如何规避**：

| Qwen 死穴 | 任务记忆的应对 |
|-----------|---------------|
| 路径死锁 | 不强锁脚本；LLM 仍可自由调 memento_create / 信息收集工具 |
| 多任务拿错剧本 | 任务级隔离 + task_id，杜绝跨任务污染 |
| 无逃生通道 | LLM 可自由放弃求助，无强制钳死 |

**核心取舍**：结构性修复（任务态隔离 + DB 硬检查）优于强制约束（tool_choice 强锁）。

### 1.9 待决开放问题

**执行脚本时的 task_id 关联**：当 LLM 调用 `execute_script` 时，怎么知道这次执行属于哪个任务？

- 活跃标记已否决（交错执行时标记无意义）
- 候选路径见 ADR-0005-EX（显式 `task_id` 传参收敛执行入口）

### 1.10 整体架构视图

```
┌─────────────────────────────────────────────────────┐
│                   skillforge mem 模块                │
│                                                      │
│  ConversationMemory     MemoryManager    TaskListManager │
│  (对话记忆)              (长期记忆)        (任务记忆) ← 新增 │
│  session_messages        memories          tasks         │
│  session_checkpoints                      task_items     │
│                                                      │
│  隔离键: user+session    user            user+session+task│
└─────────────────────────────────────────────────────┘
         │                    │                │
         └────────────────────┴────────────────┘
                              │
                        SurrealDB（同库）
```

---

## 二、核心讨论：任务记忆解决了什么？

### 2.1 多轮补充参数的问题

**没有任务记忆时**，多轮补充信息也能工作，但不可靠：

| | 纯靠 session | 有了任务记忆 |
|--|------------|-----------|
| 参数存在哪 | LLM 的上下文窗口里 | DB 的 `task_items` 表里 |
| 对话太长时 | LLM 可能忘了前面收集的参数 | DB 永远在，`memento_read` 一调就全回来 |
| 用户中途换话题再回来 | 参数可能被后续对话"冲掉" | 任务安静躺在 DB，不受影响 |
| "参数齐了没"谁判断 | LLM 自己猜 | DB：`completed_at IS NONE` 计数为 0 = 齐了 |
| 没集齐就调脚本 | 没人拦，脚本报错 | `execute_script` 的 gate 拦住，返回待办清单 |

### 2.2 不再单依赖 session

隔离维度从二维变三维：

```
以前：user_id + session_id     → 同一会话内一切共享
现在：user_id + session_id + task_id → 同一会话内任务各自独立
```

这意味着：

1. **同会话多任务不串**：用户说"帮我部署生产，顺便查一下昨天的日志" → 部署任务和日志查询互不干扰
2. **跨会话可恢复**：任务态是持久数据，session 断了、用户隔天回来，`memento_read` 照样能读回完整任务进度
3. **session 不再是任务态的唯一载体**：session 管对话历史，task_id 管任务进度，各管各的

**一句话**：session 是"我们聊了什么"，task_id 是"这个任务做到哪了"——两个正交的维度。

### 2.3 典型执行流程示例

```
用户: "帮我部署生产环境"
LLM: memento_create(skill_name="deploy", content="收集服务器地址")
LLM: "请提供服务器地址？"
用户: "10.6.6.88"
LLM: memento_update_status(task_id=1, item_id=1, done=true)
LLM: memento_create(task_id=1, content="收集部署密码")
LLM: "请提供密码？"
用户: "先帮我查一下昨天的日志"     ← 用户中途换话题了
LLM: （处理日志查询，任务态安静躺在 DB 里）
用户: "密码是 abc123"              ← 用户回来了
LLM: memento_update_status(task_id=1, item_id=2, done=true)
LLM: execute_script("deploy")      ← gate 检查：所有 item 都 done ✅ → 放行
```

**一句话总结**：任务记忆 = 把"执行到哪了、收集了哪些参数"从 LLM 的脑子里搬到数据库里，**LLM 忘了也没关系，DB 帮你记着**。

---

## 三、局限性讨论：闲聊带跑问题

### 3.1 问题场景

```
用户: "帮我部署生产环境"
LLM: 开始收集参数...
用户: "今天天气真好"          ← 闲聊
LLM: "是啊天气不错..."       ← 被带跑了，忘了部署任务
用户: "中午吃什么"           ← 继续闲聊
LLM: "推荐吃火锅"            ← 完全忘了还有任务没完成
```

任务态还在 DB 里，但 **LLM 根本想不起来去调 `memento_read`**——没人提醒它还有活没干完。

### 3.2 任务记忆能解决 vs 不能解决的

| 能解决 | 不能解决 |
|--------|---------|
| 参数不丢（存在 DB 里）| LLM 不会主动想起来去查任务 |
| 用户主动回来时能续上 | 用户不回来就一直挂着 |
| 没集齐参数时 gate 拦住 | LLM 压根没走到 execute_script |

### 3.3 真正解决"被闲聊带跑"需要加一层

ADR-0005 的 `memento_read` 是**被动**的——LLM 要主动调才能看到任务。

要解决闲聊带跑问题，需要把任务态变成**主动注入**——类似《深入理解 AI Agent》里讲的 **Agent 状态列（Agent Status Bar）**：

```
每轮对话时，框架自动在上下文末尾注入：

<agent_status>
待办任务：
  [1] 部署生产环境 (skill: deploy)
      ✅ 服务器地址: 10.6.6.88
      ❌ 部署密码: 未收集
</agent_status>
```

这样 LLM 每次生成回复时都能"瞥一眼"状态列：

```
用户: "今天天气真好"
LLM 看到状态列 → "天气不错！对了，部署生产环境还差一个密码，方便提供一下吗？"
```

### 3.4 两种机制的对比

| | 任务记忆（ADR-0005）| + Agent 状态列 |
|--|-------|-------|
| 任务态存在哪 | DB | DB + 每轮注入上下文 |
| 需要 LLM 主动查 | 是（调 memento_read）| 不需要（自动注入）|
| 闲聊带跑后能回来 | 需要用户主动提起 | LLM 自己就能看到 |
| 实现复杂度 | 低（3 个工具）| 中（需要改 context 注入逻辑）|

### 3.5 完整方案：三层防线

两者配合才是完整方案：

```
第一层：任务记忆 = 可靠的持久存储（数据不丢）
第二层：Agent 状态列 = 可靠的注意力引导（不会忘）
第三层：execute_script gate = 可靠的最终防线（没集齐就跑不了）
```
