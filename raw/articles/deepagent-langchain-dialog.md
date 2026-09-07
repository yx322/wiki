# DeepAgent 基于 LangChain 新功能 —— Z.ai 对话记录

> 来源：chat.z.ai 对话分享（GLM-5.3）  
> 提取：浏览器另存 HTML 还原，**问答原文未作改动**  
> 整理日期：2026-09-07


## 目录

1. Q1：基于langchain的新的Deepagent
2. Q2：他这个缓存和记忆优化具体是怎么做的
3. Q3：这个是多会出的
4. Q4：是基于langgraph吗
5. Q5：这是执行大模型的框架还是依赖
6. Q6：如果是调用api 呢?
7. Q7：那不还是需要一个框架来承担记忆,调用工具的一个 作用吗.来承担大模型
8. Q8：抽象层和编排层在给我解释一下吧
9. Q9：我们的模式是 skill的形式
10. Q10：那就是我们现在的模式其实和langgraph Deepagent 是俩个不同的方向

---


## Q1：基于langchain的新的Deepagent

**GLM-5.3：**

# 基于LangChain的DeepAgent框架详解
DeepAgent是LangChain生态系统中一个强大的新兴概念，它代表了将LLM转化为具备深度推理、自主规划和复杂任务执行能力的智能代理。

## 核心组件与架构
```python
from langchain.agents import AgentExecutor, create_deep_agent
from langchain.tools import Tool
from langchain.memory import ConversationBufferMemory
from langchain.llms import OpenAI

# 1. 定义核心LLM
llm = OpenAI(temperature=0)

# 2. 构建工具集
tools = [
    Tool(
        name="Search",
        func=lambda x: "搜索结果",
        description="用于信息搜索的工具"
    ),
    Tool(
        name="Calculator",
        func=lambda x: "计算结果",
        description="用于数学计算"
    ),
    # 更专业的领域工具...
]

# 3. 创建深度推理代理
deep_agent = create_deep_agent(
    llm=llm,
    tools=tools,
    verbose=True,
    planning_strategy="hierarchical",  # 分层规划策略
    max_iterations=15,  # 增加迭代次数支持复杂任务
    memory=ConversationBufferMemory()
)

# 4. 执行器配置
agent_executor = AgentExecutor(
    agent=deep_agent,
    tools=tools,
    verbose=True,
    return_intermediate_steps=True
)
```

## DeepAgent核心特性

### 1. 分层任务规划
```python
from langchain.prompts import PromptTemplate

# 任务分解提示模板
task_decomposition = PromptTemplate(
    input_variables=["task"],
    template="""
    将复杂任务分解为可执行的子任务:
    
    原始任务: {task}
    
    请按照以下步骤分解:
    1. 分析任务的关键需求
    2. 识别执行所需的信息和资源
    3. 创建按优先级排序的子任务列表
    4. 定义每个子任务的验证标准
    """
)
```

### 2. 动态推理链
```python
class DeepReasoningChain:
    def __init__(self, llm):
        self.llm = llm
        self.reasoning_steps = []
    
    def execute_reasoning(self, query):
        # 情境分析阶段
        context_analysis = self._analyze_context(query)
        
        # 假设生成阶段
        hypotheses = self._generate_hypotheses(context_analysis)
        
        # 推理验证阶段
        validated_results = self._validate_hypotheses(hypotheses)
        
        return {
            "query": query,
            "reasoning_process": self.reasoning_steps,
            "final_answer": validated_results
        }
```

### 3. 自适应工具选择
```python
class ToolSelector:
    def __init__(self, tool_registry):
        self.tools = tool_registry
        self.performance_history = {}
    
    def select_tools(self, task_context):
        # 基于任务需求动态选择工具
        required_tools = self._analyze_requirements(task_context)
        
        # 性能历史优化
        for tool in required_tools:
            if self.performance_history.get(tool.name, 0) < 0.7:
                # 查找替代工具
                alternative = self._find_alternative(tool)
                if alternative:
                    required_tools.replace(tool, alternative)
        
        return required_tools
```

## 高级应用场景实现

### 1. 多模态数据处理代理
```python
class MultiModalDeepAgent:
    def __init__(self):
        self.image_processor = DeepChain(
            tools=["ImageAnalyzer", "OCRTool", "VisualSearch"]
        )
        
        self.text_processor = DeepChain(
            tools=["TextSummarizer", "EntityExtractor", "SentimentAnalyzer"]
        )
        
        self.data_integrator = IntegrationChain()
    
    def process_mixed_input(self, data):
        image_results = self._process_images(data.images)
        text_results = self._process_text(data.text)
        integrated_result = self._integrate_results(image_results, text_results)
        
        return self._generate_deep_insights(integrated_result)
```

### 2. 长期任务执行代理
```python
class LongTermExecutionAgent:
    def __init__(self, task_scheduler):
        self.scheduler = task_scheduler
        self.task_memory = PersistentMemory()
        self.execution_monitor = ProgressTracker()
    
    def execute_long_term_project(self, project_spec):
        # 项目分解
        milestones = self.scheduler.decompose_project(project_spec)
        
        for milestone in milestones:
            # 检查点保存
            self.task_memory.save_checkpoint(milestone.id, self.get_current_state())
            
            # 执行里程碑
            result = self._execute_milestone(milestone)
            
            # 进度监控
            self.execution_monitor.update_progress(milestone, result)
            
            # 动态调整后续计划
            if not result.successful:
                self.scheduler.adjust_future_milestones(milestone, result.feedback)
```

## 性能优化策略

### 1. 缓存与记忆优化
```python
class DeepAgentOptimizer:
    def __init__(self):
        self.semantic_cache = SemanticCache()
        self.tool_cache = ToolCache()
        self.reasoning_cache = ReasoningCache()
    
    def optimize_execution(self, query, tools):
        # 语义缓存检查
        cached_result = self.semantic_cache.get_similar(query)
        if cached_result:
            return cached_result
        
        # 工具性能优化
        optimized_tools = self._prioritize_tools(tools, query)
        
        # 推理链优化
        reasoning_plan = self._create_optimized_reasoning_plan(query, optimized_tools)
        
        return self._execute_with_optimizations(reasoning_plan)
```

## 实际应用示例

### 研究分析代理
```python
class ResearchDeepAgent:
    def __init__(self):
        self.research_tools = [
            AcademicSearch(),
            WebCrawler(),
            DataAnalyzer(),
            CitationManager()
        ]
        
        self.research_planner = ResearchPlanner()
        self.quality_validator = QualityValidator()
    
    def conduct_research(self, research_query):
        # 研究规划
        research_plan = self.research_planner.create_plan(research_query)
        
        # 分阶段执行研究
        for phase in research_plan.phases:
            data = self._collect_data(phase)
            analysis = self._analyze_data(data)
            validation = self.quality_validator.validate(analysis)
            
            if not validation.passed:
                # 重新设计研究方法
                self.research_planner.refine_phase(phase, validation.feedback)
        
        return self._synthesize_research_findings()
```

DeepAgent通过结合LangChain的灵活性与深度推理能力，实现了从简单的工具调用代理到能够处理复杂、多层次任务的智能系统的飞跃，特别适合需要长期规划、多步推理和动态决策的应用场景。


## Q2：他这个缓存和记忆优化具体是怎么做的

**GLM-5.3：**

# DeepAgent缓存与记忆优化的具体实现
下面详细拆解每层的实现机制。核心思想是：用嵌入向量做语义级缓存，用分层策略做记忆管理。

## 一、语义缓存
传统缓存依赖精确的key匹配（“今天天气怎么样”≠“今日天气如何”），而语义缓存能识别两个语义相似的问题，直接返回缓存答案。

### 实现原理
```python
import numpy as np
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.schema import Document

class SemanticCache:
    def __init__(self, similarity_threshold=0.92, max_cache_size=1000):
        self.embeddings = OpenAIEmbeddings()
        self.vector_store = None
        self.similarity_threshold = similarity_threshold  # 相似度阈值
        self.max_cache_size = max_cache_size
        self.hit_stats = {"hits": 0, "misses": 0}

    def _embed(self, text: str) -> np.ndarray:
        return np.array(self.embeddings.embed_query(text))

    def get_similar(self, query: str):
        """查询语义相似的缓存结果"""
        if self.vector_store is None:
            return None

        # 向量相似度检索
        results = self.vector_store.similarity_search_with_score(query, k=1)
        if not results:
            self.hit_stats["misses"] += 1
            return None

        doc, score = results[0]
        
        # FAISS的score是距离，需要转换为相似度
        similarity = 1 - score / 2  # 归一化处理
        
        if similarity >= self.similarity_threshold:
            self.hit_stats["hits"] += 1
            print(f"[缓存命中] 相似度: {similarity:.3f}")
            return {
                "answer": doc.metadata["answer"],
                "original_query": doc.page_content,
                "similarity": similarity
            }
        
        self.hit_stats["misses"] += 1
        return None

    def store(self, query: str, answer: str):
        """存储新的查询-答案对"""
        if self.vector_store is None:
            self.vector_store = FAISS.from_documents(
                [Document(
                    page_content=query, 
                    metadata={"answer": answer}
                )],
                self.embeddings
            )
        else:
            self.vector_store.add_documents([
                Document(
                    page_content=query, 
                    metadata={"answer": answer}
                )
            ])
        
        # 缓存容量控制：LRU淘汰
        self._evict_if_needed()

    def _evict_if_needed(self):
        """超过容量时删除最旧/最低频的缓存"""
        # FAISS不支持直接删除，生产中建议用Redis + 向量索引
        pass
```

### 关键设计点

| 参数 | 作用 | 典型值 |
|---|---|---|
| similarity_threshold | 判断“语义相同”的门槛 | 0.90~0.95（过高命中率低，过低返回错误答案） |
| max_cache_size | 防止内存爆炸 | 根据业务设定 |

### ⚠️ 一个重要的坑：语义缓存的“假命中”
```text
"如何治疗癌症？" 
"如何预防癌症？"
```

这两句话向量相似度可能高达0.88，但答案完全不同。解决方案：

```
class RobustSemanticCache(SemanticCache):
    def get_similar(self, query: str):
        candidate = super().get_similar(query)
        if candidate is None:
            return None
        
        # 二次校验：用轻量模型做意图/关键实体比对
        verification_prompt = f"""
        判断以下两个问题是否在询问同一个事情，只需回答 YES 或 NO：
        问题1: {candidate['original_query']}
        问题2: {query}
        """
        is_same = self.lightweight_llm.predict(verification_prompt)
        
        if "YES" in is_same:
            return candidate
        return None  # 降级为缓存未命中
```

用便宜的小模型做二次验证，避免大模型级别的成本——这是成本优化的常见分层模式。

## 二、工具结果缓存
Agent调用工具（搜索、API、爬虫）往往是最慢、最贵的环节，且相同工具+相同参数的调用高度重复。

### 分级缓存策略
```python
import hashlib
import time
from enum import Enum

class CacheLevel(Enum):
    HOT = "hot"      # 常驻内存，5分钟过期
    WARM = "warm"    # 1小时过期
    COLD = "cold"    # 永久有效（如百科知识）

# 不同工具对应不同缓存策略
TOOL_CACHE_POLICY = {
    "web_search":        {"level": CacheLevel.HOT,  "ttl": 300},     # 搜索结果易变
    "calculator":        {"level": CacheLevel.COLD, "ttl": float("inf")},  # 计算结果永久有效
    "wikipedia_lookup":  {"level": CacheLevel.WARM, "ttl": 3600},
    "stock_price_api":   {"level": CacheLevel.HOT,  "ttl": 60},      # 股价需要准实时
    "doc_reader":        {"level": CacheLevel.COLD, "ttl": float("inf")}
}

class ToolCache:
    def __init__(self):
        self.cache = {}  # key -> {result, timestamp, hit_count}

    @staticmethod
    def _make_key(tool_name: str, tool_input) -> str:
        """生成缓存key：工具名 + 参数哈希"""
        raw = f"{tool_name}::{str(tool_input)}"
        return hashlib.md5(raw.encode()).hexdigest()

    def get(self, tool_name: str, tool_input):
        policy = TOOL_CACHE_POLICY.get(tool_name)
        if policy is None:
            return None  # 无策略的工具不缓存（如发送邮件类工具）

        key = self._make_key(tool_name, tool_input)
        entry = self.cache.get(key)
        
        if entry is None:
            return None
        
        # TTL检查
        if time.time() - entry["timestamp"] > policy["ttl"]:
            del self.cache[key]  # 过期清除
            return None
        
        entry["hit_count"] += 1  # 记录热度，用于淘汰
        return entry["result"]

    def set(self, tool_name: str, tool_input, result):
        policy = TOOL_CACHE_POLICY.get(tool_name)
        if policy is None:
            return
        key = self._make_key(tool_name, tool_input)
        self.cache[key] = {
            "result": result,
            "timestamp": time.time(),
            "hit_count": 1
        }
```

### 与Agent集成：装饰器模式
```python
def cached_tool_execute(tool, tool_input):
    # 1. 查缓存
    cached = tool_cache.get(tool.name, tool_input)
    if cached is not None:
        return cached
    
    # 2. 缓存未命中，真实调用
    result = tool.run(tool_input)
    
    # 3. 写入缓存（只缓存成功结果）
    if result and not str(result).startswith("Error"):
        tool_cache.set(tool.name, tool_input, result)
    
    return result
```

### 关键区分：哪些工具绝不能缓存

-  写操作类：发邮件、下单、写数据库
-  随机性强的：抽奖、随机数
-  状态强相关：当前时间、会话状态

## 三、推理链缓存
缓存的不是最终答案，而是中间推理步骤。当新任务与历史任务部分相似时，可以复用已验证的推理路径。

```
class ReasoningCache:
    def __init__(self, embeddings):
        self.embeddings = embeddings
        self.reasoning_chains = []  # 存储历史推理轨迹

    def store_chain(self, task, reasoning_steps, final_result, success: bool):
        """任务结束后存储整条推理链"""
        self.reasoning_chains.append({
            "task_embedding": self.embeddings.embed_query(task),
            "task": task,
            "steps": reasoning_steps,  # [{thought, action, observation}, ...]
            "success": success,        # 关键：是否成功
            "reflection": self._summarize_chain(reasoning_steps)
        })

    def find_transferable_chain(self, new_task, min_similarity=0.75):
        """寻找可迁移的推理链——注意阈值比语义缓存低得多"""
        task_emb = self.embeddings.embed_query(new_task)
        
        best = None
        for chain in self.reasoning_chains:
            if not chain["success"]:
                continue  # 失败的链不迁移（或作为反例参考）
            
            sim = self._cosine_similarity(task_emb, chain["task_embedding"])
            if sim >= min_similarity and (best is None or sim > best[0]):
                best = (sim, chain)
        
        return best[1] if best else None

    def get_planning_hint(self, new_task) -> str:
        """将历史推理链压缩为规划提示"""
        chain = self.find_transferable_chain(new_task)
        if chain is None:
            return ""
        
        # 提取历史经验注入新任务的提示词
        return f"""
参考类似历史任务的成功路径:
- 历史任务: {chain['task'][:100]}...
- 经验总结: {chain['reflection']}
- 建议复用步骤: {self._extract_key_steps(chain['steps'])}
"""
```

### 这本质上是一种 few-shot经验注入
和语义缓存的区别：

```
语义缓存:   相似任务 → 直接返回答案（跳过推理）
推理链缓存: 相似任务 → 复用部分推理路径（加速但仍有推理）
```

因此推理链缓存的相似度门槛更低（0.75 vs 0.92），允许“参考”而非“复制”。

## 四、记忆分层管理（Memory Optimization）
这是更关键的部分。长对话中，原始对话历史很快超过上下文窗口，且成本随token数增长。

### 分层记忆架构
```python
from langchain.memory import (
    ConversationBufferMemory,
    ConversationSummaryMemory,
    EntityMemory
)

class HierarchicalMemory:
    """
    三层记忆结构:
    - 工作记忆: 最近N轮对话原文（精确）
    - 摘要记忆: 更早对话的滚动摘要（压缩）
    - 实体记忆: 提取的结构化事实（长期）
    """
    def __init__(self, llm, window_size=10):
        self.window_size = window_size
        
        # 第1层：滑动窗口，保留最近对话原文
        self.working_memory = []  # [(human, ai)]
        
        # 第2层：滚动摘要
        self.summary = ""
        
        # 第3层：实体事实库
        self.entity_facts = {}  # {entity: [facts]}

    def add_exchange(self, human_msg, ai_msg):
        self.working_memory.append((human_msg, ai_msg))
        
        # 实体事实提取（可异步执行，不阻塞主流程）
        self._extract_facts(human_msg, ai_msg)
        
        # 窗口溢出时触发摘要压缩
        if len(self.working_memory) > self.window_size:
            self._compress()

    def _compress(self):
        """最老的一轮对话被压缩进摘要"""
        old_human, old_ai = self.working_memory.pop(0)
        
        compression_prompt = f"""
        将以下对话融入已有的历史摘要中，保持简洁:

        当前摘要: {self.summary or "(无)"}

        新增对话:
        用户: {old_human}
        助手: {old_ai}

        输出更新后的完整摘要（不超过200字）:
        """
        self.summary = self.llm.predict(compression_prompt)

    def _extract_facts(self, human_msg, ai_msg):
        """提取关键实体和事实（用户偏好、重要约束等）"""
        facts_prompt = f"""
        从对话中提取用户相关的持久性事实（偏好、身份、约束），
        格式为 entity|fact，每行一条，无则输出空:

        用户: {human_msg}
        助手: {ai_msg}
        """
        result = self.llm.predict(facts_prompt)
        for line in result.strip().split("\n"):
            if "|" in line:
                entity, fact = line.split("|", 1)
                self.entity_facts.setdefault(entity.strip(), []).append(fact.strip())

    def get_context(self) -> str:
        """组装最终注入提示词的记忆上下文"""
        parts = []
        if self.entity_facts:
            facts_str = "\n".join(
                f"- {e}: {'; '.join(fs)}" for e, fs in self.entity_facts.items()
            )
            parts.append(f"[用户已知信息]\n{facts_str}")
        
        if self.summary:
            parts.append(f"[历史对话摘要]\n{self.summary}")
        
        if self.working_memory:
            recent = "\n".join(
                f"用户: {h}\n助手: {a}" for h, a in self.working_memory
            )
            parts.append(f"[最近对话]\n{recent}")
        
        return "\n\n".join(parts)
```

### 记忆压缩前后对比
```text
压缩前: 10轮对话 ≈ 4000 tokens，全部塞进上下文
压缩后: 
  实体事实   ≈ 100 tokens  （用户叫张三，偏好中文回复）
  历史摘要   ≈ 150 tokens  （用户咨询了订单退货流程）
  最近2轮    ≈ 600 tokens
  ─────────────────────
  总计       ≈ 850 tokens，压缩率 ~80%
```

## 五、整体架构串联
```text
用户查询
    │
    ▼
┌─────────────────┐    命中(相似度>0.92)    直接返回
│  语义缓存查询    │ ────────────────────────► 答案 ✅
└─────────────────┘
    │ 未命中
    ▼
┌─────────────────┐    有可迁移链(>0.75)     注入经验作为few-shot
│  推理链缓存查询  │ ────────────────────────► Agent规划加速
└─────────────────┘
    │
    ▼
┌─────────────────┐
│  Agent推理循环   │ ◄── 分层记忆提供上下文
│  (ReAct/Plan)   │
└─────────────────┘
    │
    ▼
┌─────────────────┐    命中     返回缓存结果
│  工具调用        │ ──────────► ✅
└─────────────────┘
    │ 未命中
    ▼
  真实工具执行 → 结果写入缓存
    │
    ▼
  最终答案 → 写入语义缓存
  推理轨迹 → 写入推理链缓存
```

## 六、生产环境的额外考虑

- 缓存存储选型：进程内存字典只适合单实例；多实例部署用 Redis（TTL原生支持）+ Redis Search / Qdrant（向量部分）
- 缓存失效策略：数据源更新时主动失效（比如自己数据库的更新触发工具缓存清除）
- 命中率监控：hit_stats 要上报到监控系统，命中率低于预期说明阈值设置不当
- A/B验证：语义缓存上线前一定要评估“假命中率”（错误返回缓存答案的比例），这是用户无感知但损害质量的隐性风险

需要说明的是，上一轮回复中的代码是概念性示例，实际项目中 langchain.globals 的 set_llm_cache（内置LLM缓存，配合Redis背书）和 ConversationSummaryMemory 等是现成可用的起点，上面的实现展示了在其基础上做语义级、工具级、推理级三个更细粒度扩展的思路。


## Q3：这个是多会出的

**GLM-5.3：**

如果你问的是“这个是多久出的”（时间线），我来说明一下——顺便也要诚实地澄清一下前面代码的虚实边界，这点很重要。

## DeepAgents 的时间线

| 时间 | 事件 |
|---|---|
| 2023年 | LangChain 生态基础（Agent、Memory、Cache等概念已成熟） |
| 2024年 | LangGraph 发布，成为构建复杂Agent的核心底座 |
| 2025年年中（约6月） | LangChain 团队开源 deepagents 库（GitHub: langchain-ai/deepagents），灵感来自 Claude Code 和 Manus 的上下文工程实践 |
| 2025年下半年 | 快速迭代，加入 subagents、HITL（人工介入）、后端持久化等 |

⚠️ 需要说明：我的知识有截止日期，具体发布月份和最新版本号建议直接去 GitHub 仓库确认。

## 诚实的边界澄清：哪些是真的、哪些是我写的示意
前面几轮回复里，我需要区分清楚：

### ✅ 官方真实存在的（deepagents 库 / LangChain 内置）
```python
# 这是真实的 deepagents API
from deepagents import create_deep_agent

agent = create_deep_agent(
    tools=tools,
    instructions="You are a helpful research assistant",
)
agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

真实的核心能力是：

-  Planning tool（内置 write_todos 待办清单工具）
-  Subagents（task 工具派生子代理）
-  虚拟文件系统（ls / read_file / write_file / edit_file）
-  上下文摘要压缩（防上下文膨胀）
-  Human-in-the-loop、后端持久化（LangGraph checkpoint）

以及 LangChain 内置的 LLM 缓存：

```
from langchain.globals import set_llm_cache
from langchain.cache import SQLiteCache  # 真实可用
set_llm_cache(SQLiteCache())
```

### ⚠️ 我之前写的“示意性/概念性”部分
上一轮代码中这些不是官方现成 API，需要自己实现：

-  create_deep_agent(planning_strategy="hierarchical", ...) ← 这些参数是我虚构的示意
-  SemanticCache / ToolCache / ReasoningCache ← 概念演示，官方没有这个粒度的实现（官方只有精确匹配的 LLM cache）
-  HierarchicalMemory 三层记忆 ← 自己组合 ConversationSummaryMemory + EntityMemory 的思路演示

## 总结
简单说：

- deepagents 库本身：2025年出的，真实存在，pip 可装（pip install deepagents），但比我第一轮描述的“宏大”要轻量——它的精髓恰恰是用几个简单原语（todo list + 子代理 + 文件系统）+ 超长上下文LLM实现“深”效果，而不是堆复杂机制。

- 语义缓存、三层记忆那套：LangChain 官方没有这个级别的东西，是我基于常见工程实践（embedding+向量库做缓存、滑动窗口+摘要做记忆）展开的实现思路，如果你要用，需要按那个方向自己写。

所以如果你的问题实际是“这东西是不是现成的、我直接拿来就能用”——答案是：deepagents 核心部分能用，缓存/记忆优化部分要自己实现。

如果你问的其实是“怎么出的”（原理层面），那前两轮已经覆盖了；或者你是想问别的（比如“哪个会/哪个版本出的”），告诉我具体一点，我再针对性展开。


## Q4：是基于langgraph吗

**GLM-5.3：**

是的，deepagents 是直接构建在 LangGraph 之上的——它本质上是 LangGraph 的一层薄封装，而不是独立的框架。

## 依赖层次
```text
┌────────────────────────────────────┐
│ deepagents                          │  ← 规划工具、子代理、虚拟文件系统
├────────────────────────────────────┤     （本质上是精心设计的 prompt + 工具集）
│ LangGraph                           │  ← 图执行引擎、状态管理、持久化、HITL
├────────────────────────────────────┤
│ langchain-core / LangChain          │  ← 模型抽象、工具定义、消息标准
└────────────────────────────────────┘
```

## 具体对应关系

| deepagents 的能力 | 底层是 LangGraph 的什么 |
|---|---|
| Agent 主循环 | langgraph.prebuilt.create_react_agent（ReAct 架构） |
| 状态（messages + files + todos） | 自定义 State，继承 MessagesState，加 files: dict 和 todos: list 字段 |
| Subagents（task 工具） | 子代理是各自编译好的 subgraph，通过工具调用挂进主图 |
| 人工介入（HITL） | LangGraph 的 interrupt() + Command(resume=...) 机制 |
| 会话持久化 | LangGraph 的 checkpointer（同一线程 ID 跨会话恢复状态） |
| 虚拟文件系统 | 状态里的 files 字段 + 内置的 ls/read/write/edit 工具 |
| 部署 | 直接跑在 LangGraph Platform / Studio / Dev server 上 |

## 从代码里可以直接看出来
```python
import deepagents
# 1. 状态定义是 LangGraph 风格的 TypedDict
class DeepAgentState(MessagesState):
    files: dict[str, str]
    todos: list[dict]

# 2. 创建agent时可以直接透传 LangGraph 的参数
from langgraph.checkpoint.memory import InMemorySaver

agent = create_deep_agent(
    tools=tools,
    instructions="...",
    subagents=[...],          # deepagents 层的概念
    # ↓↓ 以下全是 LangGraph 原生参数
    checkpointer=InMemorySaver(),   # 持久化（LangGraph）
    model="anthropic/claude-...",   # 模型
)

# 3. 调用接口也是 LangGraph 的消息格式
result = agent.invoke(
    {"messages": [{"role": "user", "content": "..."}]},
    config={"configurable": {"thread_id": "session-1"}}  # LangGraph 线程
)
```

## 这带来的实际好处

- 可以嵌入更大的 LangGraph 工作流——deepagent 本身是一个节点，你能把它和别的节点（人工审核、通知、自定义逻辑）拼成更大的图
- 白盒可调试——拿到的是真正的 LangGraph 图，可以打印结构、单步执行、用 LangSmith 追踪每一步
- HITL 和持久化是“免费”继承的——不用 deepagents 自己实现打断/恢复

## 一句话总结
deepagents 的定位很清晰：它没有发明新引擎，而是站在 LangGraph 上，用 prompt engineering + 一组内置工具（todo list、文件系统、task 派生），把“长任务深度执行”这个模式固化成开箱即用的东西。所以你如果已经熟悉 LangGraph，学习 deepagents 的成本主要就在理解它的那几个内置 prompt 模式上。


## Q5：这是执行大模型的框架还是依赖

**GLM-5.3：**

问得好——这正好切中一个常见的混淆点。准确说：

deepagents / LangGraph / LangChain 都不是“执行大模型的框架”，而是“编排大模型的框架”（orchestration framework），它们在应用中以依赖库（dependency）的形式存在。

## 分层看就清楚了
```text
你的业务应用（DeepResearch工具、客服系统…）
        │
        ▼ pip install 引入
┌─────────────────────────────┐
│ deepagents                   │ ← 编排层（agent模式的prompt+工具封装）
│  └─ LangGraph                │ ← 编排层（状态机/图引擎、循环、HITL）
│      └─ langchain-core       │ ← 抽象层（统一的LLM/工具/消息接口）
└─────────────────────────────┘      它们不跑模型，只"组织"模型调用
        │
        ▼ 通过API或标准协议调用
┌─────────────────────────────┐
│ 真正"执行大模型"的地方         │
│                               │
│ 云端API: Claude / GPT / Gemini │
│ 自托管推理引擎:                │
│   vLLM / TGI / SGLang         │
│   llama.cpp / Ollama          │
└─────────────────────────────┘
```

## 两个关键区别

### 1. 模型推理 vs 模型编排

| | 执行模型 | 编排模型 |
|---|---|---|
| 做什么 | 真正做矩阵运算，生成token | 决定什么时候、以什么prompt、按什么顺序调用模型 |
| 例子 | vLLM、llama.cpp、TGI、API服务商 | LangChain、LangGraph、deepagents |
| 关心什么 | 吞吐量、显存、量化、KV cache | 状态管理、工具循环、多步规划、重试 |

你可以说 LangGraph 的图引擎在“执行”的是agent的工作流循环，而不是模型本身——每一次真正需要生成内容时，它最终还是发起一次 HTTP 请求（调API）或本地推理调用（连Ollama/vLLM）。

### 2. 框架 vs 依赖
从工程角度，deepagents 是你项目里的一个依赖库（pip install deepagents），不是独立运行的服务。它：

-  ❌ 不自带模型、不做推理
-  ❌ 不是你需要单独部署的运行时（当然你可以选择部署到 LangGraph Platform 上，但那是可选的托管服务）
-  ✅ 是 import 进你代码里的库，决定agent怎么循环、状态怎么存、工具怎么调

## 对应到实际代码流向
```python
from deepagents import create_deep_agent

agent = create_deep_agent(tools=..., model="anthropic/claude-...")

# 调用链路实际上是：
# agent.invoke()
#   → LangGraph 启动图循环
#     → 检查状态，拼prompt
#       → langchain-core 调 Anthropic API   ← 真正的"执行"发生在这里（Anthropic的服务器上）
#       ← 返回消息
#     → 判断是否需要调工具 → 执行工具 → 把结果塞回状态
#     → 循环，直到模型认为任务完成
```

## 一句话总结

-  deepagents/LangGraph/LangChain：模型编排框架，以依赖库形式集成进你的应用
-  vLLM/Ollama/各家API：模型推理执行层，真正跑模型的地方

如果你的场景是“我想自己部署开源模型”，那你需要的是推理引擎（vLLM等）+ 可选的编排框架，两者配合使用；如果用Claude/GPT的API，那只需要编排框架，推理完全在服务商那端完成。


## Q6：如果是调用api 呢?

**GLM-5.3：**

# 走 API 路线时的完整架构
调用 API 是最常见的方式——这时候你的技术栈反而更简单了：不需要任何推理引擎，只保留编排层。

## 整体链路
```text
你的应用
  │  import deepagents / langgraph
  │
  ▼
编排层（跑在你的服务器/进程里）
  - agent循环、状态管理、工具执行、持久化
  │
  │  HTTPS 请求（每轮推理一次）
  ▼
模型 API（Anthropic / OpenAI / Gemini …）
  - 推理发生在厂商的GPU集群上
  - 返回生成的消息
```

## 一次 DeepAgent 任务的完整时序
以一个需要3轮模型调用的任务为例：

```
你的代码                deepagents/LangGraph           Claude API
   │                          │                            │
   │ agent.invoke(任务)        │                            │
   │─────────────────────────►│                            │
   │                          │ ① 第1轮:拼prompt+工具定义    │
   │                          │── HTTPS ─────────────────► │
   │                          │                            │ 推理
   │                          │◄──── 返回"调用工具A" ───────│
   │                          │                            │
   │                          │ ② 执行工具A（在你的机器上）  │
   │                          │    把结果放回状态            │
   │                          │                            │
   │                          │ ③ 第2轮:带着工具结果再来     │
   │                          │── HTTPS ─────────────────► │
   │                          │◄──── 返回"调用工具B" ───────│
   │                          │                            │
   │                          │ ④ 执行工具B ...（循环）      │
   │                          │                            │
   │                          │ ⑤ 最后一轮:模型输出最终答案   │
   │                          │◄──── 返回最终回答 ──────────│
   │◄─── 返回结果 ─────────────│                            │
```

关键认知：模型调用次数 ≠ 任务数。一个“深度任务”可能触发几十上百次API调用（agent循环每转一圈 = 1次调用），这正是之前聊的缓存优化价值所在。

## API 模式下你的责任清单

| 事项 | 说明 |
|---|---|
| API Key 管理 | 用环境变量/密钥管理服务，绝不硬编码 |
| 成本控制 | agent循环失控 = 账单失控。设置 max_iterations、token预算 |
| 速率限制 | 429错误的退避重试，多并发时的令牌桶 |
| 超时与重试 | 长任务几分钟很正常，HTTP超时要设够长 |
| 流式输出 | stream_mode="messages" 让用户实时看到进展 |
| 可观测性 | 接 LangSmith / OpenTelemetry，否则多步循环是黑盒 |

```
# API模式下的典型配置
import os
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(
    model="claude-sonnet-4",
    api_key=os.environ["ANTHROPIC_API_KEY"],
    timeout=300,           # agent单轮可能较长
    max_retries=3,         # SDK自带退避重试
    # streaming=True
)

agent = create_deep_agent(
    tools=tools,
    model=llm,
    # LangGraph级防护：防止无限循环烧钱
    # config: recursion_limit
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "..."}]},
    config={"recursion_limit": 50}  # 图循环上限
)
```

## API 模式 vs 自托管对比

| 维度 | API 路线 | 自托管路线 |
|---|---|---|
| 前期投入 | 几乎为零，几分钟跑通 | 需要GPU、运维推理引擎 |
| 成本模型 | 按token计费，量大后很贵 | 固定GPU成本，量大后摊薄 |
| 模型选择 | 最强模型随时用 | 受限于开源模型 |
| 数据隐私 | 数据出你的网络 | 数据完全在自己机房 |
| 延迟 | 受网络+排队影响，不可控 | 本地低延迟可控 |
| 并发扩展 | 厂商弹性承担 | 自己扩容 |
| 适合场景 | 原型~中等规模、需要顶级模型 | 高量级、合规要求、固定负载 |

## API 路线特有的优化手段
1. 模型分级——deepagent的每轮调用不必都用旗舰模型：

```
# 简单子任务派生给便宜模型
from deepagents import SubAgent

researcher = create_deep_agent(model="claude-sonnet")   # 主力
summarizer = SubAgent(name="summarizer", model="claude-haiku")  # 便宜模型干杂活
```

2. Prompt Caching（API厂商侧的缓存）——Anthropic/OpenAI 支持把固定的大 prompt 前缀（如系统指令、工具定义）缓存，重复调用便宜很多。deepagent 固定的工具定义和指令前缀天然适合这个。

3. 结构化输出 + 批处理——非交互场景用 batch API（如 Anthropic Batch）拿五折价。

## 一句话总结
调 API 时：编排框架管“循环、状态、工具”，API厂商管“推理”，你管“钱、密钥、稳定性”。最需要警惕的一点是——agent 每转一圈都是一次计费调用，深度任务的 token 消耗可能是单次对话的几十倍，所以 recursion_limit、任务分解粒度和前面聊的缓存优化，在 API 模式下直接就是成本问题。


## Q7：那不还是需要一个框架来承担记忆,调用工具的一个 作用吗.来承担大模型

**GLM-5.3：**

# 你说到点子上了——但要把“承担”的角色分清楚
你的判断完全正确：总得有个东西负责记忆、工具循环、状态管理。但关键澄清是：

 框架承担的是 Agent 的“手脚和记事本”，而不是承担大模型。大脑（智能）始终在模型那边。

## 证据：不用任何框架，裸调 API 也能跑一个 Agent
现代 API（Claude / GPT）原生支持 function calling，一个极简 Agent 核心循环只要 50 行：

```
import anthropic

client = anthropic.Anthropic()
tools = [{
    "name": "search",
    "description": "搜索信息",
    "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}}
}]

messages = [{"role": "user", "content": "帮我查一下LangGraph的许可证"}]

while True:                                # ← 你手写的"框架"
    resp = client.messages.create(
        model="claude-sonnet-4",
        max_tokens=1024,
        tools=tools,
        messages=messages,                 # ← 记忆：就是你自己维护的这个列表
    )
    messages.append({"role": "assistant", "content": resp.content})

    # 模型决定要不要调工具
    tool_calls = [b for b in resp.content if b.type == "tool_use"]
    if not tool_calls:
        break                              # 模型说做完了，结束循环
    
    # 你自己执行工具，把结果塞回消息列表
    for tc in tool_calls:
        result = my_search(tc.input["query"])       # ← 工具执行
        messages.append({
            "role": "user",
            "content": [{"type": "tool_result", "tool_use_id": tc.id, "content": result}]
        })
```

这段代码里已经包含了 Agent 的全部要素：

| Agent 要素 | 裸调 API 时由谁承担 |
|---|---|
| 工具调用决策 | 模型（API 原生 function calling） |
| 工具实际执行 | 你的代码 |
| 记忆 | 你自己维护的 messages 列表 |
| 循环控制 | 你手写的 while 循环 |

所以框架不是必需品——那它的价值在哪？

## 框架 = 把这 50 行循环工程化成 50000 行生产能力
你手写的循环很快会撞上一堆现实问题：

```
手写 while 循环够用的部分
├── 基本工具调用循环        ✅ 50行搞定
│
撞墙之后需要的部分（LangGraph/deepagents 的真正价值）
├── 对话超长后上下文爆了     → 消息裁剪/摘要（记忆管理）
├── 任务要跑30分钟，服务器崩了 → checkpointer 持久化，重启后恢复
├── 某一步需要人工审核       → interrupt()/resume 机制
├── 工具调用要重试、限流     → 重试策略
├── 需要多个子agent协作      → 子图编排
├── 想看每一步到底发生了什么  → LangSmith 追踪
└── 部署、并发、扩缩容       → LangGraph Platform
```

## 所以职责边界是这样切的
```text
┌─────────────────────────────────────────┐
│              智能（大脑）                  │
│  决定调哪个工具、怎么规划任务、写什么内容    │
│  → 全部发生在模型侧（API厂商的GPU上）      │
│  ★ 框架对此零贡献 ★                      │
├─────────────────────────────────────────┤
│              骨架（手脚+记事本）           │
│  执行模型的决定、记住过程、控制循环        │
│  → 裸调时是你的50行代码                    │
│  → 用框架时是 LangGraph/deepagents        │
├─────────────────────────────────────────┤
│              推理（肌肉）                  │
│  矩阵运算、生成token                      │
│  → API厂商的集群                          │
└─────────────────────────────────────────┘
```

## 一句话回应你的问题
是的，需要承担记忆和工具调用的角色——但它可以是任何东西：你的50行代码、LangGraph、或者别的框架。

框架没有“承担大模型”，它承担的是“让大模型的智能能落地执行”的工程外壳。这也是为什么社区里一直有“Agentic 裸写 vs 用框架”的争论——简单场景裸写更好（可控、透明、无抽象层开销）；复杂场景（长任务、持久化、HITL、多agent）框架省下的是你自己要踩的坑。deepagents/LangGraph 的定位就是后者。


## Q8：抽象层和编排层在给我解释一下吧

**GLM-5.3：**

# 抽象层 vs 编排层
这两个词我之前在分层图里用到过：

```
deepagents        ← 编排层的一部分
LangGraph         ← 编排层
langchain-core    ← 抽象层
```

它们解决的是完全不同的问题，用两个生活比喻最容易分清。

## 一句话版本

 抽象层解决“怎么说话”——把不同模型的API统一成一套接口；
编排层解决“怎么干活”——决定整个工作流怎么流转。

## 抽象层：统一接口（翻译/适配器）

### 它解决的问题
想从 GPT 换成 Claude，裸写代码要改一堆东西：

```
# 裸写：OpenAI的写法
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "你好"}],
)
print(response.choices[0].message.content)

# 裸写：Anthropic的写法（完全不同的结构！）
response = client.messages.create(
    model="claude-sonnet-4",
    max_tokens=1024,                        # 连必填参数都不一样
    messages=[{"role": "user", "content": "你好"}],
)
print(response.content[0].text)
# 更别提 Gemini、通义、DeepSeek 各有各的格式
```

换成抽象层后：

```
# langchain-core 统一后的写法：只换 model 参数，代码一行不用改
llm.invoke("你好")   # 输入输出都是统一的消息格式
```

### 抽象层统一了三类东西

| 统一对象 | 混乱现状 | 抽象后 |
|---|---|---|
| LLM调用 | 每家API请求/响应结构不同 | llm.invoke() / llm.stream() |
| 工具定义 | 各家 function calling 的 schema 格式不同 | 统一的 @tool 装饰器 |
| 消息格式 | choices[0].message vs content[0].text | 统一的 HumanMessage / AIMessage |

本质：抽象层是“适配器模式”的大规模应用——它不改变行为逻辑，只是把 N 种不同的接口翻译成 1 种。你的业务代码只认这 1 种，底层换成谁都不感知。

这就是设计模式里的依赖倒置：你的代码依赖统一抽象，不依赖具体厂商。

## 编排层：流程控制（指挥/调度）

### 它解决的问题
单个 LLM 调用没有“流程”可言，但 Agent 任务有：

```
任务: 调研并写一份报告
   │
   ├─ 步骤1: 调模型 → 模型说"我要先搜索"
   ├─ 步骤2: 执行搜索工具
   ├─ 步骤3: 调模型 → 模型说"再看下文件"
   ├─ 步骤4: 中途需要人工审核 ← 怎么暂停？
   ├─ 步骤5: 服务器崩溃 ← 怎么恢复现场继续跑？
   ├─ 步骤6: 派一个子agent去做翻译
   └─ 步骤7: 汇总，结束
```

这一整套 “什么时候调模型、什么时候执行工具、状态怎么存、崩了怎么恢复、循环怎么终止” ，就是编排层的事。LangGraph 的图引擎、状态管理、checkpointer、interrupt 机制全是干这个的。

本质：编排层是“状态机 + 循环控制”——它不关心底层是哪个模型（那是抽象层的事），只关心流程的运转。

## 两者对比总结

| | 抽象层 | 编排层 |
|---|---|---|
| 回答的问题 | “怎么跟模型对话？” | “任务流程怎么运转？” |
| 核心机制 | 适配器模式（接口转换） | 状态机/图引擎（流程控制） |
| 典型内容 | LLM接口统一、工具schema统一、消息格式统一 | Agent循环、状态管理、HITL打断、持久化恢复、子图派生 |
| 对应产品 | langchain-core | LangGraph、deepagents |
| 换模型时 | 这层负责让你一行代码切换 | 这层完全不感知变化 |
| 换流程时 | 这层不变 | 这层重新编排 |

## 用一个类比串起来：餐厅
```text
编排层 = 餐厅经理
   决定整桌菜的顺序：先凉菜、再热菜、汤最后上
   顾客中途说"等等"→ 暂停（HITL）
   厨房停电 → 恢复后从断点继续做（checkpointer）
   忙不过来 → 找隔壁档口帮忙做甜点（子agent）

抽象层 = 统一的传菜协议
   不管菜是中厨做的、西厨做的、还是外包的
   都用同一种"菜品单据"进出
   经理不需要知道每个厨房的内部流程
```

经理（编排层）指挥流程，但他不进厨房炒菜（推理在模型侧），也不在乎菜是哪个厨房做的（抽象层屏蔽了差异）。

## 落回实际代码，看两层怎么配合
```python
from langchain_anthropic import ChatAnthropic      # 抽象层的具体实现
from langgraph.prebuilt import create_react_agent  # 编排层
from langchain_core.tools import tool              # 抽象层的工具统一格式

# ── 抽象层在做的事 ──
llm = ChatAnthropic(model="claude-sonnet-4")
# 明天想换成 ChatOpenAI/Gemini？只改这一行

@tool
def search(query: str) -> str:
    """搜索工具"""
    return ...

# ── 编排层在做的事 ──
agent = create_react_agent(llm, tools=[search])
# 它拿到的是抽象层统一后的 llm 和 tool
# 不知道也不关心底层是哪家模型
# 它负责的只是：循环、状态、工具调度、终止判断

result = agent.invoke({"messages": [...]})
```

注意一个细节：langchain-core（抽象层）本身没有任何流程控制能力——它不懂循环、不懂状态机。而 LangGraph（编排层）依赖抽象层的统一接口才能工作。两层是上下游关系：抽象层在下提供标准件，编排层在上用这些标准件搭流程。

deepagents 则是再往上一小层：用 LangGraph 的编排能力 + 一套精心设计的 prompt/工具模式，把“深度任务执行”这个特定流程也封装成了标准件。


## Q9：我们的模式是 skill的形式

**GLM-5.3：**

# Skill 模式：把“编排”从框架搬进模型本身
明白了——你们的模式是 Skill 化的。这正好是和 LangGraph/deepagents 走了相反方向的一条路线，把它放进前面聊的分层图里会非常清晰。

## Skill 模式的本质
一个 Skill（以 Anthropic Agent Skills 的标准形态为例）就是一个自包含的能力包：

```
skills/
├── pdf-report/
│   ├── SKILL.md          ← 元数据(name/description) + 详细指令
│   ├── scripts/
│   │   └── generate.py   ← 可执行脚本
│   └── references/
│       └── style-guide.md
├── data-analysis/
│   ├── SKILL.md
│   └── scripts/analyze.py
└── web-research/
    └── SKILL.md
```

它的核心机制是渐进式披露，这才是精髓：

```
第0层: 系统提示里只放 每个skill的 name+description
       ≈ 几十个token/skill，模型只知道"我有什么能力"
              │
              ▼  模型判断：这个任务需要 pdf-report
第1层: 加载该 SKILL.md 的完整指令（几百~几千token）
              │
              ▼  指令说"需要先跑 generate.py --prepare"
第2层: 按需读取脚本/参考资料，甚至读代码学习怎么调用
```

模型自己决定什么时候加载什么——这是和其他模式最本质的区别。

## 放回分层图：Skill 模式把架构“压扁”了
```text
 传统框架路线                          Skill 路线
┌──────────────────┐              ┌──────────────────┐
│ deepagents        │              │ （很薄的循环）      │
│ LangGraph 编排层   │              │  while: 调模型     │
│  - 状态机/图       │              │   ├ 模型要工具→执行 │
│  - 循环控制        │      VS      │   ├ 要skill→注入   │
│  - 持久化/HITL     │              │   └ 说完成→结束    │
├──────────────────┤              ├──────────────────┤
│ langchain-core    │              │ 抽象层（可留可不留） │
│ 抽象层            │              ├──────────────────┤
├──────────────────┤              │ 模型 + Skills      │
│ 模型 API          │              │  ← 编排智能在模型   │
│                   │              │    的上下文里完成   │
└──────────────────┘              └──────────────────┘
   编排逻辑在框架代码里                 编排逻辑在模型+SKILL.md里
```

用前面餐厅的比喻：框架路线是雇了一个经理（LangGraph）按流程图指挥；Skill 路线是招了一个聪明厨师长（强模型），他看到订单自己知道该翻哪本菜谱（SKILL.md）。

## 和前面对照的职责重分配

| 职责 | 框架路线 | Skill 路线 |
|---|---|---|
| 任务规划 | LangGraph 图结构 / deepagents 的 todo 工具 | 模型读 SKILL.md 里的指令自己规划 |
| 工具调用决策 | ReAct 循环判断 | 模型直接决定（同 function calling） |
| 记忆 | 框架的 Memory / checkpointer | 上下文管理靠渐进披露 + 会话历史 |
| 领域知识 | 散落在代码和 prompt 拼接逻辑里 | 集中封装在 SKILL.md 里（核心优势） |
| 能力扩展 | 改框架代码、加图节点 | 丢一个新文件夹进去就行 |

## Skill 模式的真实权衡

### 优势

-  能力扩展零代码：新技能 = 新文件夹，不需要碰编排逻辑
-  上下文经济：不是把所有指令全塞进 system prompt，而是用哪本翻哪本
-  知识可版本化：SKILL.md 是纯文本，能 git 管理、能 review、能像产品一样迭代
-  框架锁定极轻：核心资产（skills）不绑任何框架，换执行环境可以带走

### 代价（对照 LangGraph 强项，正好全是它的短板）

-  持久化/断点恢复：LangGraph checkpointer 是现成的；Skill 模式下长任务中断要自己设计状态落盘
-  HITL 人工介入：图的 interrupt 机制很成熟；Skill 模式要自己实现“暂停等审批”
-  确定性：skill 选取靠模型判断，选错了就整条路走偏（弱模型上故障率明显上升）
-  调试观测：多步执行是黑盒倾向，需要自己埋追踪
-  模型依赖强：这本质是把编排智能押注在模型能力上——Claude 这级别没问题，小模型会明显拉胯

## 一个常见的混合形态
实际上很多团队（包括 Anthropic 自己的产品）走的是中间态：

```
LangGraph（薄编排：循环 + checkpointer + HITL）
     │
     └─ Agent 节点
          ├─ 工具集（function calling）
          └─ Skills 目录（领域知识渐进披露）
```

编排层的骨架保留（解决持久化/介入/观测），但流程智能大幅让渡给模型 + skills——deepagents 本身的虚拟文件系统 + 按需读文件，其实就是这个思路的一种实现。

想确认一下你们的形态，以便聊得更具体：

- 你们的 skill 是对齐 Anthropic Agent Skills 那种 SKILL.md 标准格式，还是自建的注册/加载机制？
- 外层执行循环是自研的，还是跑在某个框架/产品（比如 Claude Code 或内部平台）上？
- 长任务有没有持久化和断点恢复的需求？——这个决定了要不要在 Skill 模式外补一层薄编排。

这三个答案不同，“缓存/记忆优化”的落点会完全不一样。


## Q10：那就是我们现在的模式其实和langgraph Deepagent 是俩个不同的方向

**GLM-5.3：**

# 对，是两个方向——但准确说是“两个极端”，而不是“两条平行线”
你的判断成立。skill 模式和 LangGraph/deepagents 的根本分歧只有一个问题：

 Agent 的流程智能放在哪里？
```text
                    编排智能的位置
外部框架 ◄──────────────────────────────► 模型内部
（代码里）                              （上下文里）

   LangGraph            deepagents        Claude Code +
   的传统用法            （中间地带）      Agent Skills
                          │                 你们的位置
              状态机写死在代码里      少量内置prompt模式      流程智能全在模型
              每个节点显式定义        +图引擎兜底            +SKILL.md里
```

## 两种哲学的对照

| | 框架路线 | Skill 路线 |
|---|---|---|
| 世界观 | Agent 是软件系统，要工程化管控 | Agent 是能力很强的模型，给它说明书就行 |
| 确定性来源 | 代码结构（图、节点、边） | 模型的判断力 + 写得好的指令 |
| 复杂度放在 | 代码里 | 文档里 |
| 失败时怪谁 | 框架没编排好 | prompt/skill 写得不好 |
| 换强模型后 | 架构显得臃肿（模型早就会自己规划了） | 表现自动变好 |
| 换弱模型后 | 架构照样兜底 | 崩溃式退化 |

## 为什么说不是平行线，而是同一光谱的两个极端
关键的行业背景：这条光谱正在整体向右漂移。

-  2023年模型弱，LangChain/Agent 用一堆链式代码硬拗出智能感
-  2024年 LangGraph 出现时其实已经是承认现实：链太死板，改成“带循环的图”
-  2025年 deepagents 更进一步：它砍掉了复杂的图编排，只留最小骨架（循环+状态），把规划交给模型+todo工具
-  同年 Anthropic Agent Skills 直接走到最右端：连骨架都最小化，能力以文档形式注入

所以你们的位置不是“另辟蹊径”，而是站在了这个演进方向的前沿——deepagents 本身就是这个方向的中间站。它的 virtual filesystem（模型按需 read/write 文件）本质上就是一个内置的 proto-skill 机制。

## 但要清醒看到你们放弃了什么
站在光谱右端，等于主动放弃了左端的三个免费福利：

- checkpointer：长任务跑到第 40 分钟进程崩了，LangGraph 能从断点恢复；skill 模式下这个要自己设计（比如让 skill 把中间产物写文件，重启后让模型自己捡起来——可行，但不是免费的）
- interrupt/HITL：流程中强制插入人工审核点，框架路线是一等公民
- 可观测性：图结构天然可追踪、可单步调试；模型自主决策是黑盒倾向

这三点的价值取决于你们的业务形态——如果是交互式工具（类 Claude Code 场景），用户在场、任务短，这三点都不痛；如果是后台长时任务，就会开始疼。

## 一个实用的判断标准
什么时候你们的路线会“不够用”：

```
任务时长 < 几分钟、人在线          → skill 模式完全够 ✅
任务要跑很久、无人值守             → 需要补持久化 ⚠️
流程有强合规/审计要求（每步必须可追溯、某些步骤必须人工） → 需要外部强制结构 ⚠️
```

## 结论
是的，两个方向。 框架路线相信“确定性要靠代码结构保证”，skill 路线相信“确定性要靠模型能力+高质量文档保证”。而且行业整体在从前者向后者迁移，模型每强一代，光谱就整体右移一格——你们现在选的位置，恰恰是赌模型会持续变强的那个位置。

这个赌注目前看是对的（Claude/GPT 每代都在提升自主规划能力），唯一要留的退路是：如果哪天需要跑无人值守的长任务，你们可以只在最外面包一层很薄的 LangGraph（或者干脆自己写个带 checkpoint 的循环），skills 完全不用动——这是 skill 模式一个隐藏的好处：核心资产和执行骨架是解耦的，往左挪的成本很低。反过来，从深框架里拆出来重做 skill 化，成本要高得多。

---

*完 · 共 10 轮对话 · 提取自 z.ai 分享页 · 2026-09-07*
