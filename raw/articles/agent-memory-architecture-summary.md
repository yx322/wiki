# Agent Memory 架构学习总结

## 一、整体目标：给 AI 增加"长期记忆能力"

普通 LLM：

```
用户问题
   |
   ↓
LLM
   |
   ↓
回答
```

问题：

一次聊天结束：

```
结束
 ↓
忘记
```

例如：

今天告诉 AI：

> 公司数据库使用 SurrealDB

一个月后：

> 公司数据库是什么？

普通模型：

> 不知道。

---

Agent Memory 系统：

增加：

```
             Agent

               |
               |

       Memory 系统

               |

          数据库存储

               |

            LLM
```

让 AI 可以：

* 记住用户信息
* 记住项目背景
* 记住历史决策
* 记住长期知识

---

## 二、核心概念：Session、Checkpoint、Memory、Cache

这是整个架构最重要的四个概念。

---

### 1. Session Message（会话消息）

#### 定义

保存真实聊天记录。

例如：

```
session_messages

msg1: 用户：你好
msg2: AI：你好
msg3: 用户：我们数据库使用 SurrealDB
msg4: AI：好的
```

特点：

* 保存完整历史
* 不做加工
* 短期使用

作用：

回答：

> 刚刚发生了什么？

---

类似人：短期记忆。

---

### 2. Checkpoint（检查点）

#### 定义

对大量聊天进行阶段总结。

例如：

100轮聊天：

```
msg1
msg2
...
msg100
```

总结：

```
checkpoint1:

用户正在开发企业AI Agent。

技术：
- Python
- SurrealDB
- RAG
- Memory系统
```

---

作用：

解决：

> 历史聊天太长，Token太多。

---

如果没有 checkpoint：

每次：

```
msg1
msg2
...
msg1000
+
问题
```

发送给模型。

问题：

* Token爆炸
* 成本增加
* 上下文污染

---

Checkpoint：

变成：

```
checkpoint: 过去1000轮总结
+
最近消息
```

---

### 3. Memory（长期记忆）

#### 定义

保存重要信息。

例如：

聊天：

```
我喜欢Python
公司使用SurrealDB
正在开发AI Agent
```

提取：

Memory：

```json
{
  "content": "用户使用SurrealDB",
  "category": "technology"
}
```

```json
{
  "content": "用户喜欢Python",
  "category": "preference"
}
```

---

作用：

回答：

> 这个用户是谁？
> 以前知道什么？

---

### 4. Cache（缓存）

Cache 不是 Memory。

区别：

|          | 作用       |
| -------- | ---------- |
| Memory   | 保存知识   |
| Checkpoint | 保存状态 |
| Cache    | 减少重复计算 |

---

Cache 在哪里？

不是数据库，而是在：

```
LLM服务端 或 推理框架
```

---

例如：

Prompt：

```
System Prompt
+
Checkpoint
+
Memory
+
问题
```

第二次前面一样，模型可以复用计算。

---

## 三、Checkpoint 和 Memory 的关系

很多人容易混淆。

关系：

```
                聊天
                  |
                  ↓
          session_messages
                  |
             100轮触发
                  |
                  ↓
                LLM
          ┌───────────────┐
          │               │
          ↓               ↓
     checkpoint        memory
     当前状态          长期知识
```

---

区别：

|    | Checkpoint       | Memory         |
| -- | ---------------- | -------------- |
| 目的 | 压缩上下文       | 保存知识       |
| 范围 | 当前Session      | 长期           |
| 变化 | 阶段更新         | 持续积累       |
| 用途 | 帮助当前聊天     | 帮助未来聊天   |

---

## 四、Checkpoint 增量机制

准确：

```
0-100轮
session_messages
      ↓
checkpoint1

101-200轮
checkpoint1
+
新增消息101-200
      ↓
checkpoint2
```

---

不是：

```
msg1-msg200 重新总结
```

原因：成本高。

---

类似 Git：

```
commit1
commit2
commit3
```

而不是每次重新提交全部代码。

---

## 五、Checkpoint 从对话生成的完整机制

### 1. Checkpoint 的本质

Checkpoint 不是简单的"复制历史"。

它是：对当前 Session 状态的**压缩表示（compressed state）**。

例如原始对话 100 轮：

```
msg1: 用户：我要做一个AI助手
msg2: 用户：准备用SurrealDB
msg3: 用户：需要RAG
...
msg100: 用户：开始设计Memory
```

完整历史太长。Checkpoint 生成后：

```
checkpoint_1:

用户正在开发企业AI Agent。

技术方向：
- SurrealDB
- RAG
- Memory系统

当前阶段：
设计Agent架构。
```

这个东西就是：未来继续聊天时，代替前100轮历史的东西。

---

### 2. 传统 Checkpoint 生成方式

假设100轮触发压缩。

流程：

```
session_messages (msg1-msg100)
        |
        ↓
Summary Prompt（"请总结下面的聊天"）
        |
        ↓
LLM
        |
        ↓
checkpoint1 → 保存到 session_checkpoint
```

缺点：

* 需要一次额外 LLM 调用
* 全部消息重新发送给模型
* Token 成本高

---

### 3. KV Cache 优化版本

目标：不想为了生成 checkpoint 再完整调用一次模型。

因为当前聊天本来就在 LLM 上下文里面。

---

#### 什么是 KV Cache？

Transformer 每处理一个 token，会计算 Key 和 Value，保存起来。

下一次不用重新计算旧内容。

例如：

```
第一次输入：A B C D
计算：KV(A), KV(B), KV(C), KV(D)

第二次输入：A B C D E
直接复用 KV(A-D)，只计算 E
```

---

#### 如何利用 KV Cache 生成 Checkpoint？

关键：不是重新调用总结模型，而是在当前对话末尾追加一个临时指令。

原来对话结构：

```
System: 你是AI助手
User: msg1
Assistant: reply1
...
User: msg100
Assistant: reply100
```

现在追加：

```
System: 你是AI助手
User: msg1
Assistant: reply1
...
User: msg100
Assistant: reply100

[临时压缩指令]
请总结以上对话，生成一个用于后续恢复上下文的checkpoint。
不要输出解释。
```

注意：这个指令不是用户消息，不是历史，只是一次运行指令。

---

#### 为什么追加在末尾？

因为 KV Cache。

前面 System + msg1-reply100 全部不变，已有 KV Cache 计算好了。

新增压缩指令，只需要计算新增 token。

所以：

```
旧上下文（KV Cache 已有）
+
新指令（只需计算新增部分）
↓
继续生成 checkpoint
```

成本最低。

---

### 4. 模型如何输出 Checkpoint？

模型看到历史 + 压缩指令，然后输出结构化总结：

```json
{
  "summary": "用户正在开发企业AI Agent",
  "technology": ["SurrealDB", "RAG"],
  "current_goal": "设计Memory架构"
}
```

这个输出就是 Checkpoint，保存到 `session_checkpoint` 表。

---

## 六、一次 Agent 请求完整流程

这是最核心。

用户：

```
Memory表怎么设计？
```

---

### 第一步：保存消息

保存到 `session_messages`。

---

### 第二步：获取上下文

#### 获取最新 checkpoint

例如：

```
用户正在设计企业AI Agent。
数据库：SurrealDB
架构：RAG + Memory
```

---

#### 检索 Memory

搜索：

```
Memory数据库
```

得到：

```
- 用户使用SurrealDB
- 用户关注RAG
- 用户正在做Agent
```

---

#### 获取最近消息

例如最近10轮：

```
用户：checkpoint是什么？
AI：...
用户：Memory是什么？
AI：...
```

---

### 第三步：Context Builder

构造最终上下文：

```
System Prompt
+
Checkpoint
+
Memory
+
Recent Messages
+
User Question
```

---

### 第四步：发送 LLM

```
LLM(prompt)
```

---

### 第五步：返回回答

---

## 七、Agent 本质是什么？

> Agent 本质不是模型，而是一个自动构建上下文的系统。

流程：

```
用户问题
↓
判断需要什么信息
↓
查询Memory
↓
查询知识库
↓
调用工具
↓
构造Prompt
↓
LLM
↓
回答
```

---

## 八、为什么 Memory 是 Prompt 构建？

> checkpoint 和 memory 都是提示词的一部分？

答案：正确。

更准确：它们属于 `Context`，最终进入 `Prompt`。

例如：

数据库中 Memory：

```
用户使用SurrealDB
```

进入 Prompt 后变为：

```
用户历史信息：
用户使用SurrealDB
```

---

所以 Memory 系统本质是：

```
动态 Prompt 构建系统
```

---

## 九、Prompt 结构

最终发送给模型：

```
SYSTEM
你是企业AI助手。

CONTEXT

[Checkpoint]
用户正在开发AI Agent。

[Memory]
- 用户使用SurrealDB
- 用户关注RAG

[Recent Messages]
最近聊天内容

USER
当前问题
```

---

## 十、为什么"末尾追加指令 + 复用缓存 + 指令不入历史"

这是高级优化。

---

### 1. 末尾追加指令

不要：

```
System
动态Memory
历史
问题
```

因为 Memory 经常变化，缓存容易失效。

---

改为：

```
稳定内容
+
动态内容放末尾
```

例如：

```
System
Checkpoint
History

------
追加：
Relevant Memory
当前任务要求
```

---

目的：提高 Prompt Cache 命中率。

---

### 2. 指令不进入历史

例如系统内部指令：

```
请回答不要超过500字
优先使用SurrealDB
```

不能保存到 `session_messages`。

否则未来 Memory 提取时可能认为用户喜欢短回答，导致污染。

---

所以保存：

```
✓ 用户消息
✓ AI回答
```

不保存：

```
✗ 内部控制指令
✗ 工具参数
✗ 临时Prompt
```

---

## 十一、Vector + BM25 + RRF 检索

Memory检索不是简单查询。采用：

### 1. Vector Search

解决语义相似问题。

例如：

问题：

```
为什么选择这个数据库？
```

找到：

```
之前选择SurrealDB原因
```

---

### 2. BM25

解决关键词匹配问题。

例如：

```
SurrealDB license
```

找到：

```
SurrealDB许可证
```

---

### 3. RRF（Reciprocal Rank Fusion）

融合多个结果。

例如：

Vector 结果：

```
SurrealDB
```

BM25 结果：

```
SurrealDB
```

Graph 结果：

```
Agent 使用 SurrealDB
```

最终：

```
SurrealDB相关记忆 排名最高
```

---

## 十二、为什么需要 Graph Memory

普通 Memory：

```
用户使用SurrealDB
用户开发Agent
用户使用RAG
```

三个孤立事实。

Graph：

```
用户
 |
开发
 ↓
AI Agent
 |
使用
 ↓
SurrealDB
 |
支持
 ↓
Vector + Graph
```

---

Graph 保存三元组：

```
Subject → Predicate → Object
```

例如：

```
用户 → 使用 → SurrealDB
```

---

## 十三、Flat KV 到 Graph Memory 演进

### 阶段1：简单 KV

```
Memory KV
- content
- category
- tags
```

---

### 阶段2：加入检索

```
Vector
BM25
RRF
```

---

### 阶段3：升级为 Graph

```
Graph Triple
Entity
Relation
Temporal
Causal Chain
```

---

## 十四、最终整体架构图

```
                    用户
                     |
                     ↓
              Agent Framework
                     |
                     ↓
              Context Builder
                     |
        ┌────────────┼────────────┐
        ↓            ↓            ↓
   Checkpoint     Memory       Session
   当前状态      长期知识      最近聊天
        ↓            ↓            ↓
        └────────────┼────────────┘
                     ↓
              Prompt Assembly
                     ↓
                   LLM
                     ↓
                  Answer
```

---

## 十五、用一句话总结整个系统

> **Session 保存发生过什么，Checkpoint 保存当前阶段状态，Memory 保存长期知识，Context Builder 把这些信息动态组装成 Prompt，让 LLM 像一个了解用户背景的智能助手一样工作。**

---

---

## 下一步：从架构理解到真正开发

1. **SurrealDB 表结构怎么设计**
2. **Memory 提取 Prompt 怎么写**
3. **Checkpoint 如何增量更新**
4. **RRF 检索代码怎么实现**
5. **Graph Memory 如何建模**

---

*整理时间：2026-07-15*
