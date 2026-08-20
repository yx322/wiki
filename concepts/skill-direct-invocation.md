---
title: Skill 直接调用与推送机制
created: 2026-08-20
updated: 2026-08-20
type: concept
tags: [skillforge, direct-invocation, push, websocket, bypass-llm]
sources: [raw/articles/skillforge-direct-invocation.md]
confidence: high
---

# Skill 直接调用与推送机制

[[skillforge]] 的两条技能调用路径：Agent 路径（经 LLM 推理）和直调路径（绕过 LLM），以及脚本向前端主动推送的机制。

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

**agent_visible: true** 时，把请求参数与结果注入 Agent 上下文并立即触发一次 Agent 执行——UI 点击后 Agent 可基于操作结果继续对话。

## 子命令设计

确定性操作实现为 Skill 的子命令，不维护独立的 actions 目录：

```
skills/cart_service/
├── scripts/run.py   # typer CLI，按子命令分发
└── SKILL.md         # 只暴露需要 LLM 感知的子命令
```

- 子命令通过 `params.argv` 列表承载，原样透传
- **不添加到 Skill 描述**（仅确定性调用时）→ 避免 LLM 加载多余内容

## 推送架构

```
┌──────────┐   POST + token    ┌────────────┐   lookup token   ┌──────────┐
│  Script  │ ─────────────────► │ SkillForge │ ───────────────► │  Client  │
│ (run.py) │                    │  (Gateway) │    via WS        │ (Frontend)│
└──────────┘                    └────────────┘                  └──────────┘
```

1. 客户端 WS 连接时生成 token，建立映射
2. SkillForge 将推送地址 + token 注入脚本环境变量
3. 脚本 POST 推送接口，携带 token 和 payload
4. SkillForge 查找映射，通过 WS 推送给前端

## 参见
- [[skillforge]] — 技能运行时框架
- [[sse-server-sent-events]] — SSE 协议
- [[websocket]] — WebSocket 协议

^[raw/articles/skillforge-direct-invocation.md]
