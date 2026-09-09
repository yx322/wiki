# Skill 表单化参数收集方案（json-render + push_form + 回调）—— 对话记录

> 日期：2026-09-07
> 主题：skill 缺参时推表单给前端，用户填写后带参回调，全程 LLM 零参与
> 状态：方案已确认（待实施）

---

## 一、需求

想做一个：**skill 执行时发现缺多参数 → 给前端返回一个表单 → 用户填写好所需参数信息后再执行调用 skill**，节省来回描述的时间。准备使用 json-render 渲染表单。

---

## 二、完整流程（已确认）

```
用户: "查一下上月订单"
    ↓ Agent 路由 / 前端直调
skill run.py 启动 → 发现缺 start_date / end_date
    ↓ push_form()                          ← push_back.py 里新增
WS 推送到前端:
{
  "type": "form",
  "callback_url": "http://host:8000/skills/order_query",
  "form_schema": { ...json-render JSON... },
  "hidden": {"order_type": "monthly"}      ← 原有参数原样带回
}
    ↓
前端 json-render 渲染表单 → 用户填 start_date / end_date → 提交
    ↓
POST /skills/order_query  (body = hidden + 用户填写的参数)
    ↓ calls.py → execute_script(params=完整参数)
skill 正常执行 → push_back(结果) / stdout 返回
```

**全程 LLM 零参与**：缺参推送、用户填写、带参回调，都是确定性通道。

---

## 三、三个改动点（都很小）

### ① `push_back.py` 加一个 `push_form()`（约 30 行）

```python
def push_form(form_schema: dict, hidden: dict | None = None) -> None:
    """推送一张 json-render 表单，用户填写后通过 callback_url 带参回调本 skill。"""
    push_back({
        "type": "form",
        "callback_url": f"{_require_env(CALLS_ENDPOINT_ENV).rstrip('/')}/{_current_skill_name()}",
        "form_schema": form_schema,
        "hidden": hidden or {},
    })
```

复用现有 `push_back` 通道和 `_current_skill_name()` 推导，**不加新端点**。

### ② 表单 schema 自动生成（关键决策点）

`core.py` 里已有 `scan_skill_interface()` 对 run.py 的 typer 应用做内省，参数名、类型、help 文本都能拿到。**自动把 typer 参数映射成 json-render schema**：

| typer 类型 | json-render 控件 |
|-----------|-----------------|
| `str` + help | text input |
| `int` / `float` | number |
| `bool` | switch |
| `datetime`/约定的 date 格式 | date-picker |
| Enum / 约定的可选值列表 | select |
| 必填（无 default） | required 校验 |

skill 作者**零成本**：照常写 typer 参数，缺参时调一句 `push_form(scan_my_form())`。个别 skill 想自定义（级联下拉等），允许传手写 schema 覆盖自动生成的。

### ③ 前端 + calls.py（基本不用改）

- 前端按 `type: "form"` 分支，json-render 渲染 `form_schema`，提交时把 `hidden` 合并进 body POST 到 `callback_url`
- `calls.py` 现有的 `execute_script(params=params, context=context)` 原样接——**后端不需要新表、不需要暂存状态**

---

## 四、给前端的完整 JSON 结构（已确认）

### 前端收到的信封

```json
{
  "type": "form",
  "user_id": "u_8a3f2c",
  "payload": {
    "type": "form",
    "callback_url": "http://host:8000/skills/order_query",
    "skill_name": "order_query",
    "form_schema": { "...json-render 协议..." },
    "hidden": { "order_type": "monthly", "dept": "sales" }
  }
}
```

### 字段职责表

| 字段 | 谁产生 | 前端拿来干嘛 |
|------|--------|-------------|
| `user_id` | **父进程**——`push_back` 从 `CONTEXT_USER_ID` 环境变量读（skill 子进程读 env），push_manager 靠它路由到该用户的 WS 连接 | 路由，前端逻辑上不用管 |
| `payload.type` | skill 脚本（`push_form` 固定写 `"form"`） | **前端分支判断**：`"form"` 走表单渲染，其他值走现有消息/表格展示 |
| `payload.callback_url` | `push_form` 自动拼（`CALLS_ENDPOINT` + skill 名） | 提交表单时的 POST 目标 |
| `payload.skill_name` | `push_form` 自动推导 | 前端埋日志/埋点用，可不展示 |
| `payload.form_schema` | skill 脚本（自动生成或手写） | **json-render 的渲染输入**，原样塞进去 |
| `payload.hidden` | skill 脚本 | **前端不渲染**，提交时原样合并进 body |

**关键点：`user_id` 是信封级的，`payload` 才是业务级。** 前端只需要处理 `payload`——和现有推送消息结构保持一致，前端已有的 `type` 分支逻辑直接扩展一个 case。

### 提交时前端回传

```json
POST http://host:8000/skills/order_query
Authorization: <用户token>
{
  "order_type": "monthly",          ← hidden 原样带回来
  "dept": "sales",
  "start_date": "2026-08-01",       ← 用户填的
  "end_date": "2026-08-31"
}
```

前端只做三件事：
1. 识别 `type === "form"` → json-render 渲染 `form_schema`（`hidden` 藏起来不显示）
2. 提交时 `body = { ...hidden, ...用户填写 }`，POST 到 `callback_url`
3. 带**用户自己的 token**（现有 access-token + client-type: pc 认证，后端 calls.py 重新走带外验证拿 user_id，**不信任表单里回传的任何身份信息**）

⚠️ **安全规则：`hidden` 里只放业务参数（查询类型、默认值），千万别把 user_id/token 放进 hidden**——JSON 在前端是明文的，身份一律走 header token 让后端自己验。

### form_schema 示例（京东 json-render 协议）

```json
{
  "type": "object",
  "title": "订单查询",
  "properties": {
    "start_date": {
      "type": "string",
      "format": "date",
      "title": "开始日期",
      "ui": { "widget": "date-picker" },
      "rules": [{ "required": true, "message": "必填" }]
    },
    "end_date": {
      "type": "string",
      "format": "date",
      "title": "结束日期",
      "ui": { "widget": "date-picker" },
      "rules": [{ "required": true, "message": "必填" }]
    },
    "dept": {
      "type": "string",
      "title": "部门",
      "ui": { "widget": "select" },
      "enum": ["sales", "finance", "warehouse"]
    }
  }
}
```

**信封结构（type/callback_url/hidden/user_id）固定，`form_schema` 里面是 json-render 自己的协议**——前端对字段名时只改 `form_schema` 内部，信封不动。

---

## 五、已拍板的设计决策

| 问题 | 决策 | 理由 |
|------|------|------|
| 缺参状态要不要暂存？ | **无状态**（hidden 字段全量回传） | 不加新表，符合"优先复用现有表/结构"原则；skill 参数通常就几个，body 大小可忽略 |
| 参数校验放哪边？ | **schema 带校验规则（前端拦截）+ calls.py 兜底校验必填项** | 前端体验好，后端防绕过直调 |
| 缺参分支的退出协议？ | **退出码 0 + stdout 输出 `{"status": "form_sent"}`** | Agent 路径收到后知道"不是失败，是在等用户填表"，对话里回"已发表单"；前端直调路径不关心 |

### 待确认项（实施前）

- json-render 的具体版本/协议分支（京东 lowcode 协议 vs 原生 render-json 协议，字段结构不一样）

---

## 六、执行细节提醒（来自 script_executor.py 现状）

1. **`parse_result` 只认 `'{"'` 开头的 stdout**——脚本如果输出 JSON 前打了日志（structlog 写 stdout），stdout 就不以 `{` 开头，整个被当原始字符串返回。所以 skill 约定：**日志走 stderr、结果走 stdout**；缺参分支退出码 0 + `{"status": "form_sent"}` 正是遵守这个约定。
2. **`build_cli_args` 的手工转义**（`\` → `\\`、`"` → `\"`）在 Windows/POSIX 混用下有边缘风险——复杂参数值（嵌套引号、换行）出问题时优先查这里；更稳的做法是参数走 `CONTEXT_JSON`/stdin 而不是 CLI 拼接。表单回调的参数经过 calls.py → execute_script 同样走这条 CLI 拼接链，**日期等简单值没问题，长文本参数要留意**。
3. **回调身份**：calls.py 从 header token 重新解析 context（走 entrance-guard API），不从 body 读身份——hidden 回传参数里即使被篡改也不影响权限。

---

## 七、实施清单（待做）

1. [ ] `push_back.py` 新增 `push_form(form_schema, hidden)` —— 复用 push_back 通道
2. [ ] typer 内省 → json-render schema 生成器（扩展 `scan_skill_interface`）
3. [ ] `calls.py` 加必填项兜底校验
4. [ ] 前端：`type === "form"` 分支 + json-render 渲染 + hidden 合并提交
5. [ ] 示例 skill：order_query（缺参推表单 → 回调执行 → push_back 结果）
6. [ ] Agent 路径处理 `{"status": "form_sent"}` 响应（回复"已发表单，请填写"）

---

*整理日期：2026-09-07 · 状态：方案确认完毕，待实施*
