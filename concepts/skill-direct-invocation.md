---
title: Skill 直接调用与推送机制
created: 2026-08-20
updated: 2026-09-14
type: concept
tags: [skillforge, direct-invocation, push, websocket, bypass-llm]
sources: [raw/articles/skillforge-direct-invocation.md]
confidence: high
---

# Skill 直接调用与推送机制

[[skillforge]] 的两条技能调用路径：Agent 路径（经 LLM 推理）和直调路径（绕过 LLM），以及脚本向前端主动推送、前端确认后回调 skill 的闭环机制。

## 两条调用路径

| 路径 | 触发 | 经过 LLM？ | 用途 |
|------|------|:---:|------|
| **Agent 路径** | 用户发消息 | ✅ | 需要理解和决策的复杂任务 |
| **直调路径** | HTTP POST / WS call | ❌ | 确定性操作（查状态、获取建议）|

## 直调路径解决的问题

1. **确定性操作的冗余开销** — 固定操作经过 LLM 增加延迟和 token 消耗
2. **缺乏直接调用能力** — 前端无法绕过 Agent 直接执行脚本
3. **服务端无法主动推送** — 脚本执行中的交互事件无法实时推送到前端
4. **推送地址可达性** — NAT/反代/容器化下脚本无法可靠获知服务端地址

## 身份链路（user_id 如何产生）

```
客户端请求（token 放 header/cookie/query，由 auth.token.extractors 配置决定）
  → extract_tokens：按 extractors 顺序提取，第一个非空即短路
  → DefaultUserResolver：拿 token 调外部 User API（auth.token.api_url）
  → _dig_path：按 auth.token.path 从响应挖出 user_id
  → context.user_id
```

- user_id 由**服务端用 token 跟 User API 换来**，前端伪造不了
- 验证失败 → user_id 为空 → WS 连接进 `_unverified` 表，收不到定向推送

## 环境变量注入

`execute_script` 执行 skill 时构造子进程 env（不污染全局，按请求隔离）：

| 环境变量 | 来源 | 用途 |
|----------|------|------|
| `CONTEXT_USER_ID` | `flatten_context_env(context)` | push_back 推送时标识目标用户 |
| `CONTEXT_JSON` | 完整 context 序列化 | 需要完整上下文的场景 |
| `CONTEXT_METADATA_*` | metadata 展平 | 如 access_token |
| `SKILLFORGE_PUSH_ENDPOINT` | `settings.server.push_url + /admin/push` | 推送接口地址 |
| `SKILLFORGE_CALLS_ENDPOINT` | `settings.server.push_url + /skills` | 回调接口前缀 |

旧变量 `SKILLFORGE_PUSH_URL` / `SKILLFORGE_CALLS_URL` 仅向后兼容回退。

## HTTP 直调

```
POST /skills/{name}
Body: {"params": {"argv": ["add"], "item_id": "100"}}
返回: {"data": {...}, "error": "", "code": 0}
```

## WebSocket 直调

```json
{
  "type": "call",
  "name": "cart_service",
  "params": {"argv": ["add"], "item_id": "100"},
  "agent_visible": false
}
```

**agent_visible: true** 时，把请求参数与结果注入 Agent 上下文（temporary 不落库）并立即触发一次 Agent 执行——UI 点击后 Agent 可基于操作结果继续对话。

## 子命令设计

确定性操作实现为 Skill 的子命令，不维护独立的 actions 目录：

```
skills/cart_service/
├── scripts/run.py   # typer CLI，按子命令分发
└── SKILL.md         # 只暴露需要 LLM 感知的子命令
```

- 子命令通过 `params.argv` 列表承载，原样透传
- **action 字段**（2026-09 新增）：`params.action` 被 execute_script pop 出拼成 typer 一级子命令，其余字段仍转 `--flag`：
  ```
  {"action": "confirm", "order": "123"}
  → python run.py confirm --order 123
  ```

## 推送架构

```
┌──────────┐  POST /admin/push  ┌────────────┐  WS send_text  ┌──────────┐
│  Script  │ ─────────────────► │ SkillForge │ ─────────────► │  Client  │
│ (run.py) │  {user_id,payload} │  (Gateway) │  按 user_id 查  │ (Frontend)│
└──────────┘                    └────────────┘  _verified 表    └──────────┘
```

1. 客户端 WS 连接 `/ws/chat`，服务端认证得到 user_id 后 `push_manager.register(ws, user_id)`，进 `_verified` 映射表
2. skill 子进程用注入的 `CONTEXT_USER_ID` + `SKILLFORGE_PUSH_ENDPOINT`，`push_back(data)` POST `{user_id, payload}`
3. 服务端从 `_verified` 表按 user_id 找 WS 连接，`send_text` 推给前端
4. 用户不在线 → 404；发送失败 → 502

## 回调闭环（push_back_call）

skill 推送时声明"确认后要执行哪个子命令"：

```python
push_back_call(params={"order": "123"}, action="confirm")
```

前端收到的消息：

```json
{
  "callback_url": "http://host:8000/skills/report",
  "payload": {"action": "confirm", "order": "123"}
}
```

- `callback_url` 由注入的 `SKILLFORGE_CALLS_ENDPOINT` + skill 名拼出（skill 名从 `sys.argv[0]` 向上两层推导，依赖 `<name>/scripts/run.py` 目录约定）
- `action` 是**作者手写的字面量**（写代码时才知道回调要执行哪个函数），不是运行时推导
- 前端确认后把 payload **原样 POST 回 callback_url**，不需要知道函数名
- action 必须对齐 typer 子命令命名：函数名转小写、`_`→`-`（`get_command_name`），或定义时 `@app.command("confirm")` 显式命名

## 参见
- [[skillforge]] — 技能运行时框架
- [[sse-server-sent-events]] — SSE 协议
- [[websocket]] — WebSocket 协议

^[raw/articles/skillforge-direct-invocation.md]
