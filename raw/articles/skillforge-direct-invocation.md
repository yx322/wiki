# SkillForge 直接调用与推送机制

> 基于 docs/skill-direct-invocation-and-push.md

---

## 一、背景与问题

当前所有技能调用都必须经过 LLM Agent 路由，存在四个问题：

1. **确定性操作的冗余开销** — 固定操作经过 LLM 增加延迟和 token 消耗
2. **缺乏直接调用能力** — 前端无法绕过 Agent 直接执行脚本
3. **服务端无法主动推送** — 脚本执行中的交互事件无法实时推送到前端
4. **推送地址可达性问题** — NAT/反代/容器化下脚本无法可靠获知服务端地址

## 二、直接调用 Skill 脚本

### 2.1 HTTP 接口

| 项 | 说明 |
|---|---|
| Method | `POST` |
| Path | `/skills/<name>` |
| Body | JSON，key-value 作为脚本参数 |
| 执行方式 | 开新进程，拼接参数调用 `scripts/run.py` |
| 返回 | `{ data, error, code }` |

**返回值格式**：
- stdout 以 `{"` 开头 → 解析为 JSON 对象
- 否则 → data 为原始字符串

**参数传递约定**：
- JSON 字段直接拼接到命令行：`--key "value"`
- 值用双引号包裹，内部转义

### 2.2 WebSocket 协议

通过 `/ws/chat` 复用聊天和直接调用两种模式：

**模式 A: Chat（流式对话）**
```json
{"messages": [{"role": "user", "content": "Hello"}], "model": "agent"}
```

**模式 B: Call（确定性调用）**
```json
{
  "type": "call",
  "name": "cart_service",
  "params": {"argv": ["add"], "item_id": "100"},
  "agent_visible": false
}
```

**响应**：单一 `result` 事件（仅作为执行确认）
```json
{"type": "result", "user_id": "alice", "error": "", "code": 0}
```

## 三、确定性操作作为 Skill 子命令

### 3.1 结构

```
skills/
└── cart_service/
    ├── scripts/run.py   # 入口脚本，按子命令分发
    └── SKILL.md         # 描述（供 LLM 读取）
```

### 3.2 子命令分离

- 子命令通过 `params.argv`（列表）承载，原样透传
- 其余 params 键转换为 `--key value` 选项参数
- **不添加到 Skill 描述**（若仅用于确定性调用）→ 避免 LLM 每次加载多余内容

### 3.3 设计说明

- **单一执行模型** — 复用同一套 execute_script
- **按需暴露** — 子命令是否进入 LLM 描述由 SKILL.md 决定
- **路由统一** — HTTP `/skills/{name}` 与 WS `type=call` 均只针对 Skill

## 四、WebSocket 推送接口

### 4.1 架构

```
┌──────────┐   POST + token    ┌────────────┐   lookup token   ┌──────────┐
│  Script  │ ─────────────────► │ SkillForge │ ───────────────► │  Client  │
│ (run.py) │                    │  (Gateway) │    via WS        │ (Frontend)│
└──────────┘                    └────────────┘                  └──────────┘
```

### 4.2 组件

| 组件 | 说明 |
|------|------|
| 推送 API | HTTP POST 端点，接收 `{token, payload}` |
| Token 映射 | 内存维护 `token → WebSocket` 连接映射 |
| 自身地址 | 配置文件中声明（`server.push_url`），环境变量注入脚本 |

### 4.3 流程

1. 客户端 WS 连接时，服务端生成 token 并建立映射
2. SkillForge 将推送地址 + token 作为环境变量注入脚本进程
3. 脚本执行时 POST 推送接口，携带 token 和消息 payload
4. SkillForge 查找映射表，通过 WS 推送给前端
5. 前端收到消息，触发 UI 操作

### 4.4 推送地址配置

```toml
[server]
push_url = "http://skillforge.internal:8000"
```

该地址通过环境变量注入脚本进程，避免因 NAT/反代/Docker 导致脚本无法回调服务端。

## 五、agent_visible 机制

WebSocket `type=call` 支持 `agent_visible` 字段：

- `false`（默认）：仅返回 result 确认
- `true`：把请求参数与结果包装为 tool_call/tool_result 注入 Agent 上下文，并**立即触发一次 Agent 执行**

用途：UI 点击触发确定性操作后，Agent 可以基于操作结果继续对话。
