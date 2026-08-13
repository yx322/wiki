---
title: KDL 与配置格式对比
created: 2026-08-13
updated: 2026-08-13
type: concept
tags: [kdl, toml, yaml, json, config-format]
sources: [raw/articles/kdl-vs-config-formats.md]
confidence: high
---

# KDL 与配置格式对比

KDL（读作 *cuddle*）是一种基于节点（Node-based）的文档语言。它的底层模型是**有向树**——一个节点天然包含四个维度：节点名称、位置参数、具名属性、子节点块。

## KDL 的核心模型

```kdl
button "提交" id="submit-btn"
```

| 维度 | 示例 | 说明 |
|------|------|------|
| **节点名称** | `button` | "我是谁"，具有最高语义统治力 |
| **位置参数** | `"提交"` | 节点的核心内容，靠位置区分 |
| **具名属性** | `id="submit-btn"` | 修饰节点的元数据，靠键名区分 |
| **子节点块** | `{ ... }` | 嵌套在花括号内的后代节点 |

## 与主流格式的哲学分野

| 格式 | 底层模型 | 语法哲学 |
|------|---------|---------|
| **KDL** | 有向树（语义树）| 节点名 + 位置参数 + 属性，一行表达完整语义 |
| **JSON** | 数据网络 | 众生平等，一切是键值对/数组，无语义主次 |
| **TOML** | 扁平键值对 | 基于节的 `键 = 值`，深度嵌套时表头冗长 |
| **YAML** | 缩进数据网 | 依赖缩进，著名的挪威问题（`NO` 变 `false`）|
| **HTML** | 标记语言 | 过于冗长，强制闭合标签 |

## 表达力对比

表达"一首带属性和标签的歌曲"：

**JSON**（臃肿）：
```json
{
  "song": {
    "title": "Hotel California",
    "attributes": { "rating": 5.0, "released": 1976 },
    "genres": ["Rock", "Classic Rock"]
  }
}
```

**KDL**（立体，一行容纳所有维度）：
```kdl
song "Hotel California" rating=5.0 released=1976 "Rock" "Classic Rock"
```

## 关键差异

| 对比维度 | KDL | JSON | 胜出 |
|----------|-----|------|------|
| 混合数据表达 | 一行内同时含语义名、纯值、键值对 | 需套多层字典 | KDL |
| 人类可读性 | 无满屏逗号/冒号/引号 | 括弧地狱 | KDL |
| 机器传输 | 解析后需转换（args/props/nodes 区分）| 全语言一行代码直接映射 | JSON |
| 架构约束 | 节点名是天然 Schema 边界 | 所有结构语法层看起来一样 | KDL |

## 底层行为特征

- **换行敏感，缩进不敏感**：一行一个节点，缩进不影响解析，`{}` 是层级的绝对权威
- **严格类型标签**（v2）：`(f64)10.0`、`(date)"2026-08-13"` 从源头规避类型误判

## 工程避坑

### 1. 列表/数组的语法隐式性

KDL 语法本身不区分"单体"还是"数组"——决定权留给代码（Pydantic/Rust 结构体）。最佳实践：加复数命名包裹节点。

### 2. 键值对不能独立成行

```kdl
// ❌ 键值对不能独立成行
database { host = "localhost" }

// ✅ 用节点名充当"键"
database { host "localhost" }
```

### 3. 位置参数 vs 键值对的代码提取

- 没有等号的纯值 → `node.args`（列表）
- 带等号的键值对 → `node.props`（字典）

### 4. 工具链

nushell 0.114.0 起内置 `from kdl` / `to kdl`，消除了日常使用最大的障碍。

## 选型判断

| 场景 | 推荐格式 | 原因 |
|------|---------|------|
| Web API、前后端传输 | **JSON** | 语言级原生映射 |
| 扁平键值为主、生态依赖成熟 | **TOML** | Cargo、Python 广泛采用 |
| Kubernetes/GitHub Actions | **YAML** | 运维生态绑定 |
| 人类手写维护的复杂树状配置 | **KDL** | 流水线声明、UI 布局、规则引擎 |

## 参见
- [[modern-language-design]] — 现代语言设计（KDL 的节点模型与语言设计哲学一致）

^[raw/articles/kdl-vs-config-formats.md]
