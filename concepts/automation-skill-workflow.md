---
title: 自动化 Skill 工作流
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [agent, automation, skills, playwright]
sources: [raw/articles/automation-skill-workflow.md]
confidence: high
---

# 自动化 Skill 工作流

将人工浏览器操作转化为可复用、可定时执行的自动化 Skill。

## 三阶段工作流

### 阶段一：开发期（交互式抓包录制）

Playwright 控制浏览器 → 人工操作 → 自动捕获 API → JSON 文件。

```python
# 核心：response 事件监听，过滤包含 /api/ 且状态码 200 的请求
page.on("response", packet_handler)
```

### 阶段二：转化期（AI 编译生成 Skill）

JSON + Prompt → Hermes → 生成统一编排器（Orchestrator）代码。

给 Hermes 的 Prompt 要求：
1. 统一对外接口 `hermes_orchestrator(username, password, ...)`
2. **主模式**：`requests.Session()` 直接请求 API（毫秒级，0% 浏览器开销）
3. **备用模式**：playwright 无头浏览器（仿真人类行为，强力兜底）

### 阶段三：生产期（双模降级执行）

```python
def hermes_orchestrator(username, password):
    try:
        result = run_with_http_client(username, password)  # 主模式
        return result
    except Exception:
        result = run_with_headless_browser(username, password)  # 备用模式
        return result
```

## 降级触发条件

- HTTP 状态码异常（非 200）
- 业务逻辑字段失败（如 `response.json().get("success") != True`）
- 超时（主模式 5 秒 timeout）

## 参见
- [[agent-compound-interest]] — Agent 复利（Skill 积累）
- [[agent-usage-patterns]] — Agent 使用模式

^[raw/articles/automation-skill-workflow.md]
