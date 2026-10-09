---
title: Auto-Compact vs 尾提示词压缩——两种上下文压缩哲学对比
created: 2026-09-30
updated: 2026-09-30
type: comparison
tags: [agent, memory, context-compression, comparison, claude-code, skillforge]
sources: []
confidence: high
---

# Auto-Compact vs 尾提示词压缩——两种上下文压缩哲学对比

解决的问题相同（上下文无限增长 vs 固定窗口），设计取舍几乎处处相反。对照双方：Claude Code 的 Auto-Compact（客户端 token 水位触发的结构化自我摘要）与 skillforge 的尾提示词 + Prefix Checkpoint（服务端数据库支撑的滚动换底）。

## 总览：两套压缩哲学

| 维度 | Claude Code Auto-Compact | skillforge 尾提示词 + Checkpoint |
|---|---|---|
| **压缩对象** | **整个 messages 数组**——压缩后数组只剩一条"前情摘要" | **只有过旧的历史段**——被压缩段替换为 checkpoint，之后的消息原样保留 |
| **触发依据** | token 水位（usage 实测，~90%） | 消息条数（`count >= 100`，粗粒度） |
| **压缩执行者** | 专门一次 LLM 调用（meta-prompt 九段式） | **搭便车**——当前 turn 顺手调 `memory_checkpoint`，不单独调 LLM |
| **压缩后的序列形状** | 推倒重来（旧前缀全灭，从零 prefill） | 平滑衔接（新前缀 = 新 checkpoint，之后继续追加） |
| **用户消息** | 九段式里近乎原文保留（第 6 段灵魂） | 全部进 checkpoint 摘要（无保真承诺） |
| **缓存视角** | 压缩 = **主动缓存雪崩**（一次性，换长跑道） | 压缩 = **前缀无缝换底**（几乎无感） |
| **失效模式** | 有损摘要 → 压缩后疯狂重读文件；滚动摘要代际漂移 | 摘要丢细节 → checkpoint 之后的原始消息还在 DB，`search_checkpoints` 可回查 |

## Auto-Compact 完整流程（对照方）

```
① 记账 ──► ② 阈值判断 ──► ③ 廉价清理(可选)──► ④ 摘要调用 ──► ⑤ 替换数组 ──► 继续干活
```

**① 记账**：每次 API 响应都带 usage 字段（input + cache_read + cache_creation + output），harness 据此持续估算上下文水位——即 Claude Code 里的 "Context left until auto-compact: X%"。

**② 阈值触发**：水位到 ~90% 触发。**为什么不等到 99%？** 留三个余量：摘要调用自己要生成输出 token；压缩后要有足够"跑道"继续干活（否则压完马上又得压）；token 估算本身有误差。

**③ 先做廉价清理**：较新版本先清旧工具调用结果（context editing）——工具输出是最大的 token 杀手（一次文件读取几千 token），换成占位符往往就能腾出大量空间，**推迟**全量压缩。全量压缩是最后手段。

**④ 摘要调用**：整段历史作为输入 + 专门的"摘要 meta-prompt"，产出结构化摘要。精妙细节：

> 这次调用的 prefill 几乎 **100% 命中缓存**：输入正好是既定前缀本身。这场"看起来最贵"的调用，实际大部分是 0.1 倍价的 cache read。

**⑤ 替换数组**：

```python
messages = [
    user_msg(f"本次会话从上一个对话继续,以下是前情摘要:\n{summary}")
]
```

旧前缀的 KV cache 全灭（小型版"改 system prompt 事故"），新请求从零 prefill——但摘要只有几千 token，重建很便宜。这就是为什么压缩要攒够阈值才做、一次做足。

## Auto-Compact 的核心：meta-prompt 九段式

摘要不是自由发挥，是强制结构：

1. **Primary Intent and Request** —— 用户的根本目标
2. **Key Technical Concepts** —— 涉及的技术概念
3. **Files and Code Sections** —— 读过/改过的文件，**带完整路径**，关键代码片段
4. **Errors and fixes** —— 踩过的坑和修复方式（防止重复犯错）
5. **Problem Solving** —— 已尝试的方案及结论
6. **All User Messages** —— **用户消息全量保留**
7. **Pending Tasks** —— 未完成事项
8. **Current Work** —— 压缩瞬间正在做什么
9. **Next Step** —— 下一步，**要求引用对话原文**

设计点：

- **第 6 段是灵魂**：用户指令是"规格说明"，意译必然丢约束（"尽量简单点"被摘要成"优化了代码"就完全变味），所以近乎原样保留
- **第 8、9 段保证无缝续接**：没有这两段，模型压缩后会"站在原地发呆"或偏离方向；要求引用原文是给续接一个硬锚点
- **第 3 段保留路径而非全文**：路径是"取回内容的钥匙"——正文扔掉，需要时重读文件即可（文件系统即记忆）

手动 `/compact 侧重数据库相关的改动` 会把侧重指令拼进摘要 prompt——压缩本身可以被引导。

### 什么被扔了，什么留了

| | 内容 | 去向 |
|---|---|---|
| **扔**（体积大头） | 工具输出、文件全文、搜索结果、命令 stdout | 只留结论/路径，需要时重读 |
| **留**（结构化摘要） | 目标、决策、错误教训、待办、当前状态 | 九段式摘要 |
| **近乎原文保留** | 用户的所有消息 | 第 6 段 |
| **根本不经过压缩** | Todo 列表、CLAUDE.md、磁盘上的代码 | **外部状态**，天然存活 |

最后一行很关键：Claude Code 的 todo 是独立于消息数组的工具状态，CLAUDE.md 和代码在磁盘上——这些"记忆"压缩后原样还在，模型随时重读。**分层记忆**：该外置的东西不占上下文，compaction 只负责清理上下文内的过程性内容。

### Auto-Compact 的失效模式（为什么压缩后 agent 总在重读文件）

- **有损性**：精确的报错信息、代码细节丢了 → 压缩后第一反应常常是重读关键文件，这是理性行为不是浪费
- **滚动摘要漂移**：长会话多次 compact，摘要是"摘要的摘要"，代际损失累积
- **"以为自己知道"**：摘要说"读过 X 文件"，模型可能跳过重读，基于记忆里的旧版本干活 → 对策是摘要里明确标注"继续前需重读的文件"

这也是文件式记忆在崛起的原因：**摘要适合记"状态"，文件适合记"事实"**——前者有损可接受，后者有损不可接受。

### 最小实现配方

```python
TRIGGER = 0.9 * context_window

# 每轮响应后记账
used = usage.input_tokens + usage.cache_read_input_tokens + usage.cache_creation_input_tokens
if used > TRIGGER:
    summary = llm_call(
        system=COMPACT_META_PROMPT,   # 九段式，强调：用户消息保留、next step 引用原文
        messages=full_history,        # 整段历史作输入
    )
    messages = [user_msg(f"前情摘要，请无缝继续:\n{summary}")]
    # todo/文件/配置等外部状态不动，天然存活
```

实践要点：压缩前先把关键结论**写入文件**（比摘要更可靠）；摘要后主动引导模型验证状态；别频繁压（每次 = 缓存全灭）。

## 三个最深的差异

### 1. "推倒重来" vs "滚动换底"——缓存账完全不同

Auto-Compact 的⑤是把数组整个换掉：

```
压缩前: [S][m1..m100]        ← 前缀缓存全在
压缩后: [S'][摘要]            ← S' 都可能变了，缓存 100% miss，从零 prefill
```

它赌的是"摘要只有几千 token，重建便宜"，且攒到 90% 才做、一次做足——雪崩是**主动支付的一次性成本**。

skillforge 的 checkpoint 换底是无感的：

```
压缩前: [ckpt_0摘要][m1..m100] [m101...]
压缩后: [ckpt_1摘要(吸收ckpt_0+前100条)] [m101...]
```

checkpoint_1 的摘要**吸收了 ckpt_0 和被压缩段的内容**，替换发生在前缀内部——严格说旧前缀的缓存也会 miss，但它每 100 条才触发一次，且换底后立刻重新稳定。**两者都付缓存重建税：Auto-Compact 攒一大笔一次付，skillforge 零花渐付。**

### 2. "摘要调用" vs "搭便车"——成本模型相反

Auto-Compact 的摘要是一次**专门的 LLM 调用**：整段历史作输入、九段式 meta-prompt 约束输出。精妙之处：这次调用的 prefill 恰好 100% 命中缓存（输入就是既定前缀），"看起来最贵"实际大半是 0.1 倍价。

skillforge 的压缩**没有独立调用**：尾提示词让当前 turn"回答用户 + 调 memory_checkpoint"一鱼两吃，摘要由模型在回答的同时顺手生成。省一次调用的代价：**摘要质量受限于当前 turn 的注意力预算**——边答问题边总结，质量天然不如专注的九段式。

### 3. 摘要结构：九段式 vs 一句话

Auto-Compact 九段式深思熟虑（用户消息保真、文件路径、pending tasks、next step 引用原文）——因为它压缩后**旧内容彻底没了**，摘要是唯一遗产。

skillforge 的 checkpoint 摘要只有"200 字概括核心内容和关键结论"——**敢这么省，是因为它不是唯一遗产**：原始消息全在 `session_messages` 表里，`search_checkpoints` 随时混合检索回查（pgvector + bm25 + RRF）。**有底气丢细节的前提是留了回查通道**——这是无状态 API + 本地数组的 Auto-Compact 做不到的。

## 选型对照

| | 选 Auto-Compact 式 | 选 skillforge 式 |
|---|---|---|
| 部署形态 | 无状态 API + 客户端持数组 | 自托管服务端 + 数据库在手 |
| 会话性质 | 超长 coding 任务（几小时、几十万 token） | 中短对话（客服/助手，几十轮） |
| 压缩后 | 任务要无缝续接，摘要是唯一记忆 | 原始数据可回查，摘要是索引 |
| 缓存策略 | 攒够一次付，接受雪崩 | 滚动换底，追求无感 |

**共同的灵魂只有一条**：不变的东西只算一次。两者的全部差异，都是"压缩"这个动作在"不可变前缀"原则上打的不同补丁——Auto-Compact 是"一次性结清"，skillforge 是"分期付款"。

## 可借鉴点

Auto-Compact **先清旧工具结果再全量压缩**（③廉价清理）值得 skillforge 抄：工具输出（如 SQL 查询返回的大表格）往往是会话里最大的 token 杀手，在触发 checkpoint 前先把老工具结果替换成占位符，能把 100 条阈值撑得更久，且不动摘要机制。

反向：skillforge 的 `search_checkpoints` 回查通道是 Auto-Compact 没有的——原始消息可检索，意味着摘要可以更激进地丢细节。

## 参见

- [[kv-cache-prompt-caching-prefill]] — 缓存雪崩与逐字节前缀匹配的原理（本页"缓存账"的依据）
- [[prefix-checkpoint]] — skillforge 侧机制的详细设计
- [[agent-loop]] — 两种压缩所在的循环语境（记忆终局写 vs 水位触发）
- [[agent-memory]] — 记忆分层全景（压缩只是其中一层）
- [[claude-code]] — Auto-Compact 的宿主（如未建页则为待建占位）
