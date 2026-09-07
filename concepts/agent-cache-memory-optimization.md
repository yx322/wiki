---
title: Agent 缓存与记忆优化
created: 2026-09-07
updated: 2026-09-07
type: concept
tags: [agent, cache, memory, semantic-cache, rag, cost]
sources: [raw/articles/deepagent-langchain-dialog.md]
confidence: medium
contested: false
---

# Agent 缓存与记忆优化

> ⚠️ 来源对话中 GLM 自己声明：以下 API 形态是**概念性示意**，LangChain 官方只有精确匹配的 LLM cache（`set_llm_cache`）和 `ConversationSummaryMemory` 等现成件。语义/工具/推理链三级缓存和三层记忆是合理的工程思路，需自行实现。

## 四级优化体系总览

```
用户查询
    ▼
语义缓存查询 ──命中(相似度>0.92)──► 直接返回答案 ✅
    ▼ 未命中
推理链缓存查询 ──有可迁移链(>0.75)──► 注入经验作为 few-shot 加速规划
    ▼
Agent 推理循环 ◄── 分层记忆提供上下文
    ▼
工具调用 ──命中──► 返回缓存结果 ✅
    ▼ 未命中
真实工具执行 → 结果写入工具缓存
    ▼
最终答案 → 写入语义缓存；推理轨迹 → 写入推理链缓存
```

## 1. 语义缓存（Semantic Cache）

传统缓存靠精确 key 匹配（"今天天气怎么样" ≠ "今日天气如何"）；语义缓存用 embedding + 向量检索识别语义等价问题。

- 相似度阈值 `similarity_threshold`：0.90~0.95（过高命中率低，过低返回错误答案）
- ⚠️ **假命中陷阱**："如何治疗癌症" vs "如何预防癌症"向量相似度可达 0.88 但答案完全不同
- 解法：**二级校验**——向量命中后用便宜的小模型做意图比对（YES/NO），成本分层
- 生产选型：进程内存字典只适合单实例；多实例用 Redis（TTL）+ Qdrant/Redis Search（向量）
- 上线前必测"假命中率"（错误返回缓存答案的比例）——用户无感知但损害质量的隐性风险

## 2. 工具结果缓存（Tool Cache）

Agent 调用工具是最慢最贵且高度重复的环节。

- 缓存 key = `hash(工具名 + 参数)`
- **分级 TTL 策略**：web_search 5min（易变）、calculator 永久、doc_reader 永久、stock_price 60s
- 装饰器模式集成：查缓存 → 未命中真实调用 → 只缓存成功结果
- **绝不能缓存的工具**：写操作类（发邮件/下单/写库）、随机类、状态强相关（当前时间）

## 3. 推理链缓存（Reasoning Cache）

缓存的不是最终答案而是**中间推理步骤**，新任务与历史部分相似时复用已验证的推理路径。

- 任务结束存储整条链：`{task_embedding, steps[], success, reflection}`
- 只迁移成功链（失败链可作为反例参考）
- 相似度门槛 **0.75**（比语义缓存低得多）：允许"参考"而非"复制"
- 用法：把历史链压缩为规划提示注入新任务（few-shot 经验注入）

与语义缓存的区别：

```
语义缓存:   相似任务 → 直接返回答案（跳过推理）
推理链缓存: 相似任务 → 复用部分推理路径（加速但仍有推理）
```

## 4. 分层记忆（Hierarchical Memory）

长对话原文很快撑爆上下文。三层结构：

| 层 | 内容 | 成本 |
|---|---|---|
| 工作记忆 | 最近 N 轮对话原文 | 精确但贵 |
| 摘要记忆 | 更早对话的滚动摘要 | 压缩 |
| 实体记忆 | 提取的结构化事实（偏好/约束） | 长期 |

- 滑动窗口溢出 → 最老一轮被 LLM 压缩进摘要（滚动式，非全量重算）
- 实体提取可异步执行不阻塞主流程
- 压缩效果示例：10 轮 ≈ 4000 tokens → 事实 100 + 摘要 150 + 最近 2 轮 600 ≈ 850 tokens（**压缩率 ~80%**）

与 skillforge 的 [[prefix-checkpoint]] 同源：都是"窗口内保原文 + 窗口外进不可变摘要"，但 skillforge 用 checkpoint 不可变设计兼顾 KV cache 命中。

## 5. API 模式下的成本优化

- **模型分级**：主力任务用旗舰模型，杂活（摘要等）派给便宜模型（SubAgent 机制）
- **Prompt Caching**：固定前缀（系统指令、工具定义）厂商侧缓存，deepagent 的固定前缀天然适合
- **Batch API**：非交互场景五折价
- **护栏**：`recursion_limit` 防循环失控、token 预算、429 退避重试——agent 每转一圈都是一次计费调用，深度任务 token 消耗可能是单次对话的几十倍

## 参见

- [[kv-cache]] — 推理引擎侧的 KV cache（与本页应用层缓存不同层）
- [[agent-memory]] — Agent 记忆两层架构
- [[prefix-checkpoint]] — skillforge 的会话压缩实现
- [[deepagents]] — 对话中的主体框架
- [[hnsw-index]] — 语义缓存的向量检索底层
- [[rrf-fusion]] — 混合检索（语义缓存可结合）

^[raw/articles/deepagent-langchain-dialog.md]
