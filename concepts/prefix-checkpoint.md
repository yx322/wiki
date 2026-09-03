---
title: Prefix Checkpoint 机制
created: 2026-08-20
updated: 2026-08-20
type: concept
tags: [skillforge, memory, kv-cache, checkpoint, compression]
sources: [raw/articles/skillforge-memory-system.md]
confidence: high
---

# Prefix Checkpoint 机制

[[skillforge]] 对话记忆的核心机制——通过**不可变的 checkpoint** 实现上下文压缩，同时对 KV cache 友好。

## 核心问题

LLM 的上下文窗口有限，长对话需要压缩。但压缩不能破坏 KV cache——否则每次压缩后所有缓存失效，性能崩塌。

## 解决方案

```
消息追加 → 累积到阈值(100条) → 下一个用户提问时注入尾提示词
  → Agent 一次 turn 同时完成：
    1. 回答用户问题（正常输出）
    2. 调用 memory_store 提取长期记忆（LLM 主动）
    3. 调用 memory_checkpoint 压缩为摘要（LLM 被动）
  → checkpoint 不可变，永久可缓存
```

## 为什么不可变是关键

| 方式 | KV Cache 行为 |
|------|-------------|
| **Agno 滑动窗口** | 每次从 DB 读最近 N 条，N 不断变化 → **缓存完全失效** |
| **Prefix Checkpoint** | checkpoint 一旦写入永远不变 → **可永久缓存** |

```
KV cache 中：[checkpoint_0] + [msg_101..msg_150]
                                        ↓ 阈值到达
压缩后：[checkpoint_1] + [msg_151..msg_160]
```

checkpoint_0 永远不变，下次访问同一 session 时可直接从缓存读取。

## 两种记忆的分工

| 记忆类型 | 触发方式 | 谁生成内容 | 用途 |
|----------|----------|-----------|------|
| **长期记忆**（memory_store）| **主动**：LLM 判断"值得记住" | LLM 决定存什么 | 跨会话知识，语义检索 |
| **短期记忆**（memory_checkpoint）| **被动**：阈值触发 | LLM 生成对话摘要 | KV cache 优化，上下文压缩 |

## 架构铁律

**mem 模块厚，agent 适配层薄。**

```python
# 适配层只做三步（无任何记忆逻辑）
wrapper.run(user_message)
  ① memory.push_user_message(session_id, user)     # 缓冲
  ② agent.additional_input = memory.get_context()   # 注入 checkpoint + 增量
  ③ memory.write_assistant_message(session_id, reply) # 写入 DB
```

记忆的存储、检索、压缩逻辑全部在 mem 模块内部，不依赖框架。迁移到其他框架只需重写适配层。

## 参见
- [[skillforge]] — 技能运行时框架
- [[kv-cache]] — KV Cache 与前缀缓存
- [[agent-memory]] — Agent 记忆两层架构

^[raw/articles/skillforge-memory-system.md]
