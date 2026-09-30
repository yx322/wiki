---
title: Agent 循环（Tool-Call Loop）
created: 2026-09-29
updated: 2026-09-29
type: concept
tags: [agent, llm, skillforge, architecture]
sources: []
confidence: high
---

# Agent 循环（Tool-Call Loop）

Agent 循环用 skillforge 引擎讲最实在——它就在 `engine/llm.py` 的 `_run_loop_sync` 里，130 行，比任何教科书画图都清楚。

## 本质：一个 while 循环

LLM 本身是**无状态的**：一次请求进去，一次回答出来，就结束了。它不会"接着干活"、不会"记住刚才"、更不会主动用工具。所谓 Agent，就是**在模型外面套一个循环**，把"用工具"变成它的可选项：

```
用户消息进 messages
    ↓
┌─► while True:
│     ① 带 messages + tools 调 LLM（流式）
│     ② 模型回答分两种：
│        ├─ 纯文本 → 终局：落记忆、emit 完成、return
│        └─ 要工具 → 收集 tool_calls
│     ③ 逐个执行工具，结果作为 {role:"tool"} append 进 messages
│     ④ 回到 ①（模型看到工具结果，决定继续调工具还是回答）
└──────┘
```

为什么必须是循环：**模型每轮只能"决策一步"**。查订单要先调 `query_db`，拿到结果才能决定要不要再调 `format_table`，拿到表格才能写总结——每一步的下一步取决于上一步的工具结果，这个依赖链天然是循环，不是一次请求能表达的了。

对应到代码就四个动作：

```python
while True:
    stream = client.chat.completions.create(model, messages, tools=tool_schemas, stream=True)
    # ...流式累积 content + tool_calls...
    if not tool_call_accum:
        # 终局：无工具调用 = 模型认为活干完了
        self._session_memory.write_assistant_message(session_id, content)  # 只在终局写记忆
        yield RunCompletedEvent(content=content)
        return
    messages.append({"role": "assistant", "content": content, "tool_calls": tool_calls})
    for tc in tool_calls:
        result = self._execute_tool(tc["function"]["name"], tc["function"]["arguments"], ...)
        messages.append({"role": "tool", "tool_call_id": tc["id"], "content": result_str})
    # 循环回去——messages 变长了，模型这轮能看到工具结果
```

循环不变量就一条：**`messages` 数组不断变长**。assistant 说"我要调工具"、工具说"结果是这个"、assistant 看着结果接着说——全部追加进同一个数组，每轮整体重发给模型。模型的"连续性"不是状态，是重放的历史。

## 循环上挂着的五个工程问题

裸循环谁都会写，Agent 框架的含金量全在循环周围：

**1. 终止条件——不能信任模型自己停**

模型可能陷入"调工具→不满意→再调→再调"的死循环。skillforge 的解法：`max_tool_steps=20`，且**只数 `run_skill_script` 的轮次**（`skill_rounds`）——memory_search 这种轻工具不占配额，真正烧资源的脚本执行才有上限。到顶强制注入"达到最大工具轮数上限，停止"并退出。

**2. 流式下的工具调用拼装**

流式响应里 `tool_calls` 的 name/arguments 是**按 index 分片多次到达的**，必须边 emit 边累积：

```python
tool_call_accum: dict[int, dict] = {}
if tc.function and tc.function.arguments:
    call_args_accum[idx] = call_args_accum.get(idx, "") + tc.function.arguments
```

用户看到的流式文本（RunContentEvent 逐片转发）和内部累积的完整工具调用是**并行的两条线**——这也是为什么 assistant 的 content 可以流式给用户，工具调用却必须等流结束才能执行。

**3. 错误也要喂回给模型**

工具执行失败不是抛异常打断循环，而是把错误**作为 tool 结果写回**：

```python
return json.dumps({"error": f"Unknown tool: {name}", "available_tools": [...]}, ...), True
```

模型看到 error + available_tools 清单会自己纠正（换个工具、改参数）——**错误是循环的燃料，不是循环的终点**。真正要打断循环的只有两类：LLM 请求本身失败（429/网络，透传给用户）和轮数超限。

**4. 记忆只在终局写**

`write_assistant_message` 只在 `not tool_call_accum` 分支调用——中间那些工具轮次（assistant 说要调工具、工具回结果）**不落库**。落库的是完整的一轮对话：用户的原始问题和最终回答。中间过程由 messages 数组在本次循环内自持。这避免了"半截工具链"污染下次会话的上下文。

**5. 注入点循环前一次性完成**

system prompt（含 `<skills_system>` 目录）、ConversationMemory 的 checkpoint+增量、agent_visible 确定性调用——这些在 `arun()` 进入循环**之前**就组装进 messages。循环内只有 LLM 调用和工具执行两件事，不掺记忆逻辑——这就是 mem 厚、适配层薄的铁律在循环上的体现。

## 抽象一层：所有 Agent 框架的循环都是这个形状

```
while 未终止:
    response = llm(messages, tools)
    if response 需要工具:
        results = 执行工具(response.tool_calls)   # 并行/串行是策略
        messages += [assistant, tool_results]
    else:
        return response
```

差异只在填空题：

| 填空 | LangGraph | skillforge | Claude Code |
|---|---|---|---|
| 终止条件 | 图走到 END 节点 | 无工具调用 / max_steps | 无工具调用 / 上下文预算 |
| 工具集 | 节点绑定的 tools | skill 三工具 + memory_tools + memento | 文件/bash/搜索等 |
| 状态 | checkpointer 落库 | messages 数组 + 会话记忆 | 上下文窗口 |
| 循环主体 | 节点函数 | while + tool 回填 | while + tool 回填 |

LangGraph 把循环展开成显式的图（节点=步，边=转移），换来确定性兜底和断点恢复；skillforge 选了薄循环，把确定性交给模型判断力+skill 文档质量——这正是 [[skill-vs-framework]] 那条光谱在"循环实现"上的投影。**循环本身没有秘密，秘密在循环每一圈注入了什么、拦下了什么。**

## 和前几个话题的连线

- 循环的**每圈成本** = 全量 messages 重发 → 所以才有 [[prefix-checkpoint]]（不可变前缀换 KV cache 命中）和 Fluxen ADR 0005（token 级 patch 替代整树重发）——**"别重发没变的部分"是同一个原理在 LLM 上下文和 UI 渲染两个域的投影**
- 循环的**每圈燃料** = 工具结果 → 所以工具结果的质量（错误信息带 available_tools、schema 校验拦住非法输出）直接决定循环收敛速度
- 循环的**出口** = 模型判断"干完了" → 这个判断本身不可靠，所以需要 max_steps 这类外部保险丝

## 参见

- [[skillforge]] — 循环的宿主引擎（ADR-0006 自研 tool-call 循环）
- [[skill-vs-framework]] — 薄循环 vs 图编排的光谱定位
- [[prefix-checkpoint]] — 循环每圈成本的优化（不可变前缀）
- [[skill-direct-invocation]] — 绕过循环的确定性通道
- [[task-memory-ticket]] — 循环内脚本执行前的任务记忆 gate
