---
title: KV Cache 与前缀缓存
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [kv-cache, llm, performance]
sources: [raw/articles/agent-memory-architecture-summary.md, raw/articles/agent-memory-architecture.md]
confidence: high
---

# KV Cache 与前缀缓存

Transformer 每处理一个 token，会计算 Key 和 Value 并保存。下一次不用重新计算旧内容。

## 基本机制

```
第一次输入：A B C D
计算：KV(A), KV(B), KV(C), KV(D)

第二次输入：A B C D E
直接复用 KV(A-D)，只计算 E
```

**前提**：前缀必须完全一致。哪怕改一个字，从那个字开始后面的 KV Cache 全部失效。

## 对 Agent 的影响

### Checkpoint 为什么 cache 友好

checkpoint 一旦写入**不可变**，后续所有 turn 共享同一个 checkpoint 文本。LLM API 的 prompt caching 将 checkpoint 部分缓存在 GPU 显存中，只有增量部分每次变化。

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]    ← ckpt 部分走 KV cache
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: 达到阈值 → 压缩为 ckpt_1 → 重置
```

### 全量注入为什么 cache 不友好

全量注入每轮把所有历史注入 prompt，内容每轮变化，KV cache 失效。

### 滑动窗口为什么 cache 不友好

Agno 的 `add_history_to_context` 每次从 DB 读最近 N 条完整消息。随着新消息到来，N 条的组成不断变化（旧的被挤出、新的被加入），KV cache 每轮失效。

## 首 Token 延迟 (TTFT) 实验

保持系统提示词不变，连续发起两次对话：
- 第二次首 token 延迟明显更低（前缀缓存命中）

修改系统提示词开头的任意几个字符，再发起一次：
- 首 token 延迟回到初始水平（整个前缀需要重新计算）

## 压缩时的缓存利用

阈值到达时，当前 prompt 已包含 `[ckpt] + [msg_101..msg_150]`，全部在 KV cache 中。此时在末尾追加压缩指令（如"分析以上对话，输出 checkpoint 和 memories"），LLM 以缓存的完整上下文处理这条新指令，成本极低。

## Prompt 缓存友好的结构

1. **System prompt**（固定）→ 缓存命中
2. **Few-shot 示例**（固定）→ 缓存命中
3. **Checkpoint**（不可变）→ 缓存命中
4. **增量消息**（每次变化）→ 唯一的计算部分

System prompt + few-shot + checkpoint 占 prompt 总量的 60-80%。缓存命中后，实际计算成本 ≈ 20-40%。

## 参见
- [[agent-memory]] — 记忆架构
- [[llm-fundamentals]] — LLM 基础认知

^[raw/articles/agent-memory-architecture-summary.md]
^[raw/articles/agent-memory-architecture.md]
