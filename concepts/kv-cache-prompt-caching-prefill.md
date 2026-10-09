---
title: KV Cache 与 Prompt Caching 与 Prefill
created: 2026-09-30
updated: 2026-09-30
type: concept
tags: [kv-cache, llm, performance, inference]
sources: []
confidence: high
---

# KV Cache 与 Prompt Caching 与 Prefill

三次递进对话的完整沉淀：KV Cache（单次生成内）→ Prompt Caching（跨请求）→ Prefill（推理两阶段）。三者是理解一切 LLM 推理优化的钥匙。

## 一、KV Cache：为什么需要它

Transformer 生成文本是**自回归**的：一次生成一个 token。生成第 t+1 个 token 时，需要用它的 Query 去和**前面所有 token 的 Key、Value** 做注意力计算。

朴素做法的问题：每生成一个 token，都把整个序列重新过一遍模型，重新计算所有历史 token 的 K 和 V——但这些值明明上次就算过了，完全没变。

**KV Cache 的核心思想**：历史 token 的 K、V 一旦算出就不再变化，把它们缓存起来，每步只计算**新 token** 的 Q、K、V。

> 为什么只缓存 K 和 V，不缓存 Q？因为注意力只需要"新 token 的 Q"和"所有 token 的 K、V"。历史 token 的 Q 在它们各自的步骤里用完就丢，之后再也不需要了。

### 工作流程

```
Prefill（预填充）阶段：
  输入整个 prompt → 并行计算所有位置的 K、V → 存入缓存

Decode（解码）阶段，每步循环：
  1. 输入最新一个 token
  2. 只计算这个 token 的 Q、K、V
  3. 新的 K、V 追加到缓存
  4. 用新的 Q 和缓存中全部 K、V 做注意力
  5. 输出下一个 token
```

伪代码对比：

```python
# 无缓存：每步重算整个序列，复杂度 O(n²)/步
for _ in range(n):
    logits = model(all_tokens)
    all_tokens.append(sample(logits))

# 有缓存：每步只算新 token，复杂度 O(n)/步
past_kv = None
for _ in range(n):
    logits, past_kv = model(last_token, past_kv=past_kv)
    last_token = sample(logits)
```

### 显存开销有多大？

每个 token 缓存的大小：

$$2 \times \text{层数} \times \text{KV头数} \times \text{head\_dim} \times \text{精度字节数}$$

以 LLaMA-2 7B（FP16，MHA）为例：
- 32 层 × 32 头 × 128 head_dim × 2 字节 × 2（K 和 V）= **每 token 512 KB**
- 序列长度 4096 → **单条请求约 2 GB！**

这就是长上下文推理显存吃紧、batch 扩不开的根本原因——KV cache 往往比模型权重本身还占显存。

### 常见优化技术

| 技术 | 思路 |
|------|------|
| **GQA / MQA** | 多个 Q 头共享一组 KV 头，缓存缩小数倍到数十倍（LLaMA-3、Mistral 都在用） |
| **MLA**（DeepSeek） | 把 KV 压缩到低维隐空间，缓存极小且精度损失小 |
| **KV 量化** | 用 FP8/INT8 存 K、V，直接砍半或砍 3/4 |
| **PagedAttention**（vLLM） | 像操作系统分页一样管理显存，消除碎片、支持前缀共享 |
| **Prefix Caching** | 多个请求共享相同前缀的 KV（如同一个 system prompt） |
| **滑动窗口** | 只保留最近窗口内的 KV（StreamingLLM 等） |

### 一个关键特性：Decode 是访存瓶颈

每生成一个 token，都要把**整个 KV cache 从显存读一遍**，而计算量只有一点点（一个 token 的矩阵乘）。所以 decode 阶段 GPU 算力利用率很低，是 **memory bandwidth bound** 的。

这解释了很多现象：
- 为什么连续 batching 能大幅提升吞吐（一次前向处理多个请求，摊薄访存开销）
- 为什么 KV 压缩对推理加速如此重要
- 为什么预填充可以并行、逐 token 生成却难以在序列维度并行

## 二、Prompt Caching：KV Cache 的"跨请求"版本

- **KV Cache**：在一次生成**内部**复用——同一次请求里不重算历史 token
- **Prompt Caching**：在**多个请求之间**复用——不同请求共享的 prompt 前缀，其 KV cache 只算一次，存下来供后续请求直接用

本质就是把某段 prompt 的 KV cache 持久化一段时间，实现"不变的内容只算一次"。

### 为什么收益巨大

LLM 推理分两段：prefill（处理整个输入，计算量 O(n²)，长 prompt 是延迟大头，100K token 可能要几秒）和 decode（逐 token 生成）。

而现实中 prompt 里有大量**每次都一样**的内容：

- 固定的 system prompt
- 工具/函数定义
- few-shot 示例
- RAG 检索的长文档
- **多轮对话历史**（每轮都在上一轮基础上追加，天然前缀递增）

Agent 应用一个请求动辄带几万 token 的"固定开销"，每次都重新 prefill 纯属浪费。

### 工作机制

```
首次请求：完整 prefill → 前缀 KV 写入缓存（按"写入价"计费，通常略有加价）

后续请求：与缓存做前缀匹配
  → 命中部分直接复用 KV，跳过 prefill（按"读取价"计费，大幅折扣）
  → 只对新增的后缀做 prefill

闲置一段时间 → 缓存淘汰（TTL）；再次命中通常会刷新 TTL
```

各家的具体策略（数字以官方文档为准）：

| 平台 | 触发方式 | 最小长度 | 命中价 | 写入价 | 有效期 |
|------|---------|---------|--------|--------|--------|
| Anthropic | 显式 `cache_control` 断点 | 1024+（部分模型 2048） | 输入价 ×0.1 | ×1.25 | 5 分钟，命中刷新 |
| OpenAI | 全自动，无需标记 | 1024 | ×0.5 | 无加价 | 闲置 5–10 分钟淘汰 |
| Gemini | 隐式自动 / 显式 API | 数千 token | ×0.25 + 存储费 | — | 可配置，默认 1 小时 |

### 为什么必须"从头精确匹配"

这一点直接呼应 KV cache 的**位置依赖性**：第 i 个 token 的 K/V 是它前面所有 token 的函数。哪怕只改了 prompt 开头一个字，后面所有位置的 KV 全部作废。

由此得出一条黄金法则：

> **静态内容放前面，动态内容放后面。**

时间戳、用户 ID、随机数如果拼在 prompt 靠前的位置，整个缓存就废了。

### API 用法示例

```python
response = client.messages.create(
    model="claude-sonnet-4-5",
    system=[{
        "type": "text",
        "text": LONG_SYSTEM_PROMPT,   # 几千上万 token 的固定内容
        "cache_control": {"type": "ephemeral"}   # 打一个缓存断点
    }],
    messages=[...],
)
# 响应 usage 中：
#   cache_creation_input_tokens → 本次新写入缓存的 token
#   cache_read_input_tokens     → 命中缓存的 token
```

OpenAI 是全自动的，看 `prompt_tokens_details.cached_tokens` 即可；Gemini 看 `usageMetadata.cachedContentTokenCount`。

**多轮对话是最大受益者**：第 N 轮的完整历史正好是第 N-1 轮的前缀，追加一条新消息，之前全部轮次的 KV 都能命中。

### 自托管视角

自己部署模型时同样有对应实现：

- **vLLM**：automatic prefix caching——paged KV cache 按 block 内容哈希寻址，相同前缀的 block 直接物理共享
- **SGLang**：RadixAttention——用基数树管理所有请求的前缀，自动共享、LRU 淘汰
- **关键配套：cache-aware 路由**——缓存在具体 GPU 实例的显存里，相同前缀的请求必须路由到同一台机器才能命中，所以需要前缀感知的负载均衡
- **进阶**：LMCache、Mooncake 把 KV 卸载到 CPU/远端存储，延长缓存寿命、支持跨节点迁移

自托管时收益不只是省钱：省下的是 prefill 的算力和显存带宽，同样的硬件能扛明显更高的吞吐。

### 实践建议

1. **结构固定**：system prompt → 工具定义 → 文档 → few-shot → 用户输入，动态内容永远放最后
2. **前缀逐字节一致**：空格、标点、消息顺序、JSON 键序的细微变化都会破坏匹配
3. **攒够长度**：太短不值得缓存，也够不到最小阈值
4. **一次性内容别打断点**：写入有加价，没人复用就亏
5. **多轮对话追加，不要重写历史**
6. **监控命中率**：像优化数据库缓存一样，盯着 usage 里的 cached tokens 字段
7. **注意发版**：改一版 system prompt = 全体用户缓存瞬间失效

### 层面澄清：不是框架层，是服务端推理基础设施层

这是常见误解。它不在你的应用代码里，不在 LangChain 之类的框架里，也不改模型本身——它存在于 **API 提供方的机房里，在推理引擎/调度层**。

```
你的应用代码 / 框架        ← 只负责把 prompt "排好序"（静态前缀 + 动态后缀）
      ↓ HTTP 调用
API 服务层（Anthropic 等）  ← 缓存断点管理、命中匹配、计费、TTL、路由到缓存所在节点
      ↓
推理引擎（vLLM / 自研）     ← KV cache 管理、前缀匹配、显存调度
      ↓
GPU 显存                   ← 缓存实际存放的地方
```

关键推论：**换什么框架完全不影响缓存**。用 LangChain、LlamaIndex 还是裸 `curl` 调 API，缓存照常工作，因为匹配和存储都发生在 Anthropic 的服务器上。`cache_control` 只是你在请求里"声明一个断点"，真正的匹配、存储、失效全在服务端。

### 那个故事为什么成立：system prompt 改版引发缓存雪崩

流传很广的事（大意如此，细节以当事人分享为准）：Anthropic 工程师改了 system prompt 里的一处措辞，结果推理成本和延迟显著上升。机制拆开看：

1. **system prompt 是"完美缓存前缀"**——所有用户的所有请求，开头都是同一段 system prompt，命中率接近 100%
2. **改一个字 = 前缀匹配全盘失效**——前缀哪怕只差一个 token，后面所有位置的缓存全部作废，没有"部分命中"
3. **缓存雪崩**：

```
改版前：大多数请求 → 命中缓存 → prefill 开销很小
改版瞬间：所有请求 → 全量 miss → 每个 token 都要重新 prefill
         ↓
prefill 是 compute-bound，计算量瞬间暴涨
→ GPU 过载、请求排队、TTFT（首 token 延迟）全面上升
→ 看起来就是"推理变慢了"
```

之后新前缀的缓存逐步重建，速度才恢复回来。所以准确说是**改版后的过渡期变慢变贵**，不是永久变慢。

4. **反直觉的事实**：Prompt caching 不只是"省钱功能"，而是生产系统的**性能支柱**。Claude 敢给每个请求塞上万 token 的 system prompt 和工具定义，赌的就是"反正命中缓存"。缓存一失效，这个前提塌了。

两个旁证：Anthropic 把 system prompt 设计成独立参数（不混在 messages 里）——就是为了让它成为干净稳定的缓存前缀，这本身是围绕缓存做的 API 设计；官方文档反复强调"逐字节一致"——匹配发生在 token 序列级别，是精确匹配，不是 embedding 相似度。

## 三、Prefill 详解：两种完全不同性质的计算负载

prefill 和 decode 的对比是理解一切 LLM 推理优化的钥匙：

| | Prefill | Decode |
|---|---|---|
| 处理对象 | 整个 prompt（n 个 token）一次性 | 每次 1 个 token |
| 计算形态 | 矩阵×矩阵（GEMM） | 矩阵×向量（GEMV） |
| 瓶颈 | **compute-bound（算力）** | **memory-bound（显存带宽）** |
| 并行性 | 全序列并行，和训练前向一样 | 严格串行，只能靠 batch 摊薄 |
| 产出 | 全部 KV cache + 第一个输出 token | 之后每一个 token |
| 延迟指标 | TTFT（首 token 延迟） | ITL/TPOT（每 token 间隔） |

### Prefill 内部发生了什么

对 n 个 token 的 prompt，模型只做**一次前向传播**：

- 所有位置并行计算 Q、K、V——因果掩码保证每个位置只看前面，所以可以像训练时一样整个序列一起算
- 注意力用下三角掩码一次算完整个 n×n 分数矩阵
- 输出两样东西：全部位置的 KV（写入 cache）+ 最后位置的 logits（采样出第一个 token）

计算量（量级估算）：

$$\underbrace{2Pn}_{\text{线性层，随 }n\text{ 线性}} + \underbrace{O(n^2)}_{\text{注意力}}$$

常被忽略的细节：**每个 token 的 prefill 成本不相等**。注意力项是 n²，所以 prompt 越长，后面新增的每 1K token 比开头的 1K 更贵。短上下文时线性层主导，几万 token 以上注意力项才反超。

### 为什么是 compute-bound：算术强度视角

GPU 有个"平衡点" = 峰值算力 ÷ 显存带宽。H100 约 1000 TFLOPS ÷ 3.35 TB/s ≈ **300 FLOP/字节**——从显存每读 1 字节权重，得做约 300 次浮点运算才能让算力单元吃饱。

- **Prefill 一次算 n 个 token**：权重只从显存读一遍，却做 2Pn FLOPs → 算术强度 ≈ n。n 到几百，GPU 就被算力喂饱 → compute-bound
- **Decode 每步只算 B 个 token**（B 是 batch 大小）：同样把全部权重读一遍，只做 2PB FLOPs → 算术强度 ≈ B。B=1~几十，远低于平衡点 → 每一步都在"等显存"

这也是 API 定价中 **input 比 output 便宜好几倍**的根本原因：prefill 并行效率极高（单卡每秒能吞上万个输入 token），decode 单流每秒只能吐几十个，单位服务成本完全不同。

### 一个直观的数字

7B 模型 + H100、10K token prompt（粗略数量级）：

- Prefill：约 0.5 秒上下 → 等效 **上万 token/秒**
- Decode 单流：约 50~150 token/秒

同样"处理 1 万个 token"，prefill 快约两个数量级。而这 0.5 秒对用户就是按下回车后、看到第一个字之前的死等——**prefill 时间 ≈ TTFT**，直接决定产品的"响应感"。流式输出帮不上这段：第一个 token 出来之前，没东西可流。

### 调度问题：大 prefill 会砸到别人头上

在 continuous batching 系统里，一个 100K token 的 prefill 若整块执行，会独占 GPU 数秒，期间**所有正在 decode 的请求全部停摆**——用户看到的就是"打字突然卡住"。

```
整块 prefill:  [===100K prefill，数秒===][d][d][d][d]...
                   ↑ 其他请求的 decode 全卡在这

切块后:       [块1+d][块2+d][块3+d][块4+d]...
                   ↑ 每个时间片都匀给 decode 一点
```

**Chunked prefill** 就是解法：把 prompt 切成 512~2048 token 的块，每步调度一块，与 decode 请求混在同一个 batch。附带一个精妙的好处（Sarathi 论文的核心洞察）：decode 是 memory-bound、算力大量闲置，把它的小计算塞进 prefill 的大 GEMM 里，闲置算力被"免费"利用。代价是 prefill 自身效率略降。vLLM v1 默认开启，SGLang/TensorRT-LLM 均支持。

### 更彻底的方案：PD 分离

既然两种负载瓶颈不同，干脆拆到不同的机器池：

```
请求 → Prefill 节点（堆算力，FLOPS 打满）
          ↓ 传输 prompt 的 KV cache
       Decode 节点（堆显存带宽/容量，持续吐 token）
```

- 关键代价是 **KV 迁移**：7B·FP16 每 token 512KB，10K prompt 就是 ~5GB，必须走 NVLink/RDMA 级互联，否则还不如在 decode 节点重新 prefill
- 生产实例：Mooncake（月之暗面/Kimi）、DistServe、DeepSeek 的部署架构、vLLM/SGLang 的 disagg 模式、NVIDIA Dynamo

### 优化手段全景

| 手段 | 一句话 | 解决什么 |
|---|---|---|
| Prompt/prefix caching | 命中则**整段跳过 prefill** | 最大头 |
| Chunked prefill | 切块混批调度 | 不阻塞他人，平滑 ITL |
| FlashAttention | IO-aware 注意力 kernel | 长序列 prefill 本身提速 |
| 并行 prefill | 超长 prompt 切多卡算完再拼 KV | 极致压 TTFT |
| PD 分离 | 专用机器池各干各的 | 集群吞吐与延迟兼得 |

### 闭环：回到那个故事

现在 Anthropic 那件事每一环都清楚了：命中缓存 = 跳过 prefill（又快又省）→ 改一个字 = 前缀全 miss → 所有请求回退到全量 prefill → prefill 是 compute-bound 的大规模矩阵乘 → 瞬间吃满 GPU 算力 → 请求排队、TTFT 飙升 → "推理变慢"。

---

**一句话总结**：KV cache 让单次生成不必重算历史，prompt caching 让多次请求不必重算共享前缀——哲学一脉相承：不变的东西，只算一次。prefill 是"一次算完整个 prompt"的并行阶段——compute-bound、效率高、但贵在量大；decode 是"一步一步吐字"的串行阶段——memory-bound、低效但无法避免。几乎所有推理优化都是在这两种负载之间做文章：**缓存让 prefill 少算，chunked 让它不打扰别人，PD 分离让它们各干各的。**

## 参见

- [[kv-cache]] — 本页第一节的基础版（skillforge 语境下的 KV Cache 机制）
- [[prefix-checkpoint]] — 同一"不可变前缀"思想在 skillforge 会话记忆上的应用
- [[llm-fundamentals]] — 自回归生成的底层心智模型
- [[traditional-vs-moe-models]] — 推理负载的另一种分法（稠密 vs MoE）
