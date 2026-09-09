---
title: Skill 表单化参数收集（push_form + json-render + 回调）
created: 2026-09-07
updated: 2026-09-07
type: concept
tags: [skillforge, push, form, json-render, callback, parameter-collection]
sources: [raw/articles/skillforge-form-param-collection-dialog.md]
confidence: high
contested: false
---

# Skill 表单化参数收集

skill 缺参时直接推一张表单给前端，用户填写后带参回调，**全程 LLM 零参与**——缺参推送、用户填写、带参回调都是确定性通道。基于现有 `push_back_call` → 前端 → `calls` 直调链路串成闭环，json-render 只负责"表单怎么画"。

## 流程

```
skill run.py 启动 → 发现缺 start_date / end_date
    ↓ push_form()                          ← push_back.py 新增
WS 推送: {type:"form", callback_url, form_schema, hidden}
    ↓
前端 json-render 渲染 → 用户填写 → 提交
    ↓
POST /skills/order_query  (body = hidden + 用户填写)
    ↓ calls.py → execute_script(params=完整参数)
skill 正常执行 → push_back(结果)
```

## 前端收到的信封（固定不变）

```json
{
  "type": "form",
  "user_id": "u_8a3f2c",
  "payload": {
    "type": "form",
    "callback_url": "http://host:8000/skills/order_query",
    "skill_name": "order_query",
    "form_schema": { "...json-render 协议..." },
    "hidden": { "order_type": "monthly" }
  }
}
```

| 字段 | 谁产生 | 用途 |
|------|--------|------|
| `user_id` | 父进程（CONTEXT_USER_ID env） | push_manager 路由 WS，前端不用管 |
| `payload.type` | `push_form` 固定 `"form"` | 前端分支判断 |
| `payload.callback_url` | `push_form` 自动拼（CALLS_ENDPOINT + skill 名） | 提交 POST 目标 |
| `payload.form_schema` | skill 脚本（自动生成或手写） | json-render 渲染输入 |
| `payload.hidden` | skill 脚本 | 前端不渲染，提交时合并进 body |

**user_id 是信封级（路由用），payload 才是业务级。**

## 提交回传

```json
POST /skills/order_query
Authorization: <用户token>
{ "order_type": "monthly", "start_date": "2026-08-01", "end_date": "2026-08-31" }
```

⚠️ **安全规则**：`hidden` 只放业务参数，**绝不放 user_id/token**——JSON 在前端是明文。身份一律走 header token，calls.py 重新走带外验证（entrance-guard API），不信任回传数据里的任何身份信息。

## schema 自动生成：typer 内省 → json-render

`core.py` 已有 `scan_skill_interface()` 对 run.py 的 typer 应用内省。映射规则：

| typer 类型 | json-render 控件 |
|-----------|-----------------|
| `str` + help | text input |
| `int` / `float` | number |
| `bool` | switch |
| date 格式 | date-picker |
| Enum / 可选值 | select |
| 无 default（必填） | required 校验 |

skill 作者零成本：照常写 typer 参数，缺参时调 `push_form(scan_my_form())`；个别 skill 可传手写 schema 覆盖。

## 设计决策（已拍板）

| 问题 | 决策 | 理由 |
|------|------|------|
| 缺参状态暂存？ | **无状态**，hidden 全量回传 | 不加新表，复用现有结构 |
| 参数校验？ | schema 带规则（前端拦截）+ calls.py 兜底必填 | 前端体验 + 防绕过直调 |
| 缺参退出协议？ | 退出码 0 + stdout `{"status": "form_sent"}` | Agent 知道是"等填表"不是失败 |

## 实施要点（来自 script_executor.py 现状约束）

- **日志走 stderr、结果走 stdout**——`parse_result` 只认 `'{"'` 开头的 stdout，混入日志整个变原始字符串
- **回调参数走 CLI 拼接链**（calls.py → execute_script → build_cli_args）——日期等简单值没问题，长文本参数留意转义边缘风险
- **无状态设计**：hidden 把原参数带回，回调即全新完整调用，后端无需暂存

## 参见

- [[skill-direct-invocation]] — 直调与推送双通道（本方案的通道基础）
- [[skillforge]] — 技能运行时
- [[task-memory-ticket]] — 任务记忆（多步参数收集的另一种思路：LLM 逐轮收集 vs 表单一次收集）

^[raw/articles/skillforge-form-param-collection-dialog.md]
