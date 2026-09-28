---
title: Daft 分布式多模态 DataFrame 库
created: 2026-09-14
updated: 2026-09-14
type: entity
tags: [polars, lakehouse, iceberg, performance, architecture]
sources: []
confidence: high
---

# Daft 分布式多模态 DataFrame 库

开源分布式 DataFrame 库（[getdaft.io](https://www.getdaft.io)），Rust 引擎 + Python API，主打**分布式执行（Ray）+ 多模态数据（图像/张量/嵌入向量）**。PyPI 包名 `getdaft`，导入 `import daft`。

## 与 Polars 的定位分野

同为 Rust 引擎 + Python API，API 风格相近（学习迁移成本低），但定位不同：**Polars 主打单机极致性能，Daft 主打分布式 + 多模态**。

| 维度 | Polars | Daft |
|------|--------|------|
| 执行模式 | 单机多线程（即时 + 惰性） | 本地多线程，一行 `set_runner_ray()` 切 Ray 集群 |
| 数据规模 | 单机内存（流式引擎支持超内存，上限仍是单机） | 本地或分布式集群 |
| 数据类型 | 表格数据（list/struct/时间序列丰富） | 原生多模态：Image/Tensor/Embedding 列类型 |
| 纯表格性能 | 第一梯队 | 不错但通常略逊 |
| I/O 密集任务 | 一般 | 强（`download()` 自动并行/重试/限流） |
| 成熟度 | 非常成熟 | 较新，快速迭代中 |
| GPU | cudf.polars 加速 | GPU UDF |

**一句话**：Daft ≈「Polars + 分布式 + 多模态」，代价是纯表格场景生态与性能细节不如 Polars。

## 多模态的本质：引擎"认识"你的数据

图像/张量/嵌入是**一等公民的列类型**（`DataType.image("RGB")`、`DataType.embedding(float32, 512)`），带模式/宽高/形状元数据——对比 pandas/Polars 里图片只能是 bytes 大对象，引擎完全不知道里面是什么。

由此带来的能力：

1. **声明式多模态流水线**：`col("url").download().image.decode().image.resize(224,224)` 三步在引擎内融合成一条流水线——下载、解码、重采样无中间落盘，并发/重试/限流引擎托管。pandas 手搓等价功能要自己写线程池、重试循环、内存管理
2. **谓词下推对非结构化数据生效**：`where(label=="cat").select(col("url").download()...)` 只有过滤后的图片会被下载——过滤在 I/O 之前生效，省带宽省 CPU
3. **GPU UDF 批处理托管**：`@daft.udf(batch_size=64)` 包装 CLIP 等模型，批处理/设备搬运/GPU 节点调度引擎管理
4. **直通 PyTorch**：`df.to_torch()` / `iter_torch_batches(batch_size=256)`，图像保持 numpy/torch 兼容格式，免写 Dataset 类

## 生态与 I/O

Parquet/CSV/JSON/Delta Lake/**Apache Iceberg**/Lance 读写；本地路径与 S3/GCS/Azure/OSS 等云存储统一 URL 语法。

**与 Iceberg 的关系对项目有直接参考价值**：Daft 原生支持 Iceberg catalog 读表，写路径比 Polars 成熟（Polars `write_iceberg()` 在本项目已确认与阿里云 OSS Tables 协议不兼容，见 [[polars-iceberg-oss-tables]]）。湖仓读侧用 Daft 替代 Polars 是低成本选项；写侧如需绕开 OSS Tables 协议问题，Daft 的 Iceberg 集成值得一试。

## 选型

| 场景 | 推荐 |
|------|------|
| 单机放得下的表格分析/ETL | **Polars** |
| 追求单机性能极限 | **Polars** |
| 图像/音频/嵌入等多模态管道 | **Daft** |
| 数据量大到要集群（不想用 PySpark/JVM） | **Daft** |
| ML 数据准备（下载→预处理→喂 PyTorch） | **Daft** |

## 与本项目 Polars 用法的关系（选型结论，2026-09-14 查证官方文档后修正）

当前湖仓链路（[[task-data-pipeline]]）中 Polars 承担查湖客户端 + 轻量转换：纯表格数据、单机量级、单进程部署——Polars 全面占优，**不迁移**。

**查证修正**：Daft 的 Iceberg 集成**基于 PyIceberg**（`load_catalog` 即 PyIceberg 函数，`write_iceberg` 接收 PyIceberg Table），并非独立 Rust-native 实现——因此 OSS Tables 协议不兼容问题（[[polars-iceberg-oss-tables]]）在 Daft 上原样存在，"绕开写入问题"的预期不成立。

| Polars 现有角色（逐文件核实） | Daft 能替？ | 结论 |
|---|---|---|
| asw sync.py：ETL + 写 Delta Lake（`DeltaTable.merge` upsert） | ⚠️ Daft 读 Delta 但无 merge 等价，只 append/overwrite | 替了要重写 upsert，不值 |
| query_oss.py：DuckDB 结果 `.pl()` → markdown 给 LLM | ✅ 随便替 | 无收益，纯换口味 |
| search-todo sync.py：批量同步 ETL（cast/空值填充） | ✅ API 近同构 | 收益≈0 |
| 湖读主链路 | ❌ 实际主力是 pyiceberg + DuckDB，Polars 只在两端 | Daft 无位置 |

**关键查证**：Daft 的 Iceberg 读路径**尚不应用 V2 equality deletes**（官方 FAQ：on the roadmap）——而 RW iceberg upsert sink 恰以 merge-on-read 写入 equality delete 文件，直接读出重复行。项目手写的 [[iceberg-reader|IcebergDedupReader]]（按主键取最大 seq 去重）解决的正是这个问题，Daft 当前同样需要这层处理，故读侧也无优势。

**Daft 的真实进场时机**（未来）：
1. 其读路径支持 V2 equality deletes 后 → `daft.read_iceberg()` 可替代手写 dedup reader
2. 数据量大到单机放不下 → Ray 分布式
3. 出现图像/嵌入管道 → 多模态算子

三条目前均不满足，定位保持"地图点位，不入栈"。

## 参见

- [[polars]] — 单机高性能对照物
- [[iceberg]] — Daft 原生支持的表格式
- [[polars-iceberg-oss-tables]] — Polars 写 OSS Tables 的协议兼容性问题（Daft 同受影响）
- [[lakehouse]] — 湖仓一体架构
- [[hnsw-index]] — Embedding 列的下游消费（向量检索）
