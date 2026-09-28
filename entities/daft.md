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

开源分布式 DataFrame 库（[getdaft.io](https://www.getdaft.io)），Rust 引擎 + Python API（PyO3 绑定），主打**分布式执行（Ray）+ 多模态数据（图像/张量/嵌入向量）**。PyPI 包名 `getdaft`，导入 `import daft`。官方仅提供 Python API，无独立 Rust crate。

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
| GPU | cudf.polars 加速 | GPU UDF（自动 batching/设备调度） |

**一句话**：Daft ≈「Polars + 分布式 + 多模态」，代价是纯表格场景生态与性能细节不如 Polars。

## 多模态的本质：引擎"认识"你的数据

图像/张量/嵌入是**一等公民的列类型**（`DataType.image("RGB")`、`DataType.embedding(float32, 512)`），带模式/宽高/形状元数据——对比 pandas/Polars 里图片只能是 bytes 大对象，引擎完全不知道里面是什么。

由此带来的能力：

1. **声明式多模态流水线**：`col("url").download().image.decode().image.resize(224,224)` 三步在引擎内融合成一条流水线——下载、解码、重采样无中间落盘，并发/重试/限流引擎托管。pandas 手搓等价功能要自己写线程池、重试循环、内存管理
2. **谓词下推对非结构化数据生效**：`where(label=="cat").select(col("url").download()...)` 只有过滤后的图片会被下载——过滤在 I/O 之前生效，省带宽省 CPU
3. **GPU UDF 批处理托管**：`@daft.udf(batch_size=64)` 包装 CLIP 等模型，批处理/设备搬运/GPU 节点调度引擎管理
4. **直通 PyTorch**：`df.to_torch()` / `iter_torch_batches(batch_size=256)`，图像保持 numpy/torch 兼容格式，免写 Dataset 类

## Iceberg 集成：基于 PyIceberg（2026-09 查证官方文档）

**关键事实**：Daft 的 Iceberg 集成**构建在 PyIceberg 之上**——`load_catalog()` 即 PyIceberg 函数，`df.write_iceberg(table)` 接收 PyIceberg Table 对象。官方表述 "natively integrated with PyIceberg"。

- **读**：分布式 I/O + 谓词下推（分区裁剪、min/max 文件剪枝）+ branch/tag/snapshot 读取——这部分是 Daft 引擎层加的价值
- **写**：append / 全表 overwrite / 静态分区 overwrite（`overwrite_filter`，带写入校验）；**不支持 upsert/copy-on-write 更新**
- **不支持 V2 equality deletes 的应用**（官方 FAQ：on the roadmap）——读 merge-on-read 表会把更新过的行读成多份
- **catalog 能力 = PyIceberg 的 catalog 能力**：PyIceberg 连不上的 catalog（如阿里云 OSS Tables，见 [[polars-iceberg-oss-tables]]），Daft 同样连不上

Delta Lake 同样支持读写（URI 直连 S3/GCS/Azure），但同样无 merge/upsert 语义。

## 选型

| 场景 | 推荐 |
|------|------|
| 单机放得下的表格分析/ETL | **Polars** |
| 追求单机性能极限 | **Polars** |
| 图像/音频/嵌入等多模态管道 | **Daft** |
| 数据量大到要集群（不想用 PySpark/JVM） | **Daft** |
| ML 数据准备（下载→预处理→喂 PyTorch） | **Daft** |

## 与本项目 Polars 用法的关系（选型结论）

当前湖仓链路（[[task-data-pipeline]]）中 Polars 承担查湖客户端 + 轻量转换：纯表格数据、单机量级、单进程部署——Polars 全面占优，**不迁移**。

Polars 现有角色逐文件核实：

| Polars 现有角色 | Daft 能替？ | 结论 |
|---|---|---|
| asw sync.py：ETL + 写 Delta Lake（`DeltaTable.merge` upsert） | ⚠️ Daft 读 Delta 但无 merge 等价，只 append/overwrite | 替了要重写 upsert，不值 |
| query_oss.py：DuckDB 结果 `.pl()` → markdown 给 LLM | ✅ 随便替 | 无收益，纯换口味 |
| search-todo sync.py：批量同步 ETL（cast/空值填充） | ✅ API 近同构 | 收益≈0 |
| 湖读主链路 | ❌ 实际主力是 pyiceberg + DuckDB，Polars 只在两端 | Daft 无位置 |

两个关键查证：

1. **OSS Tables 写入问题在 Daft 上原样存在**——其 catalog 层就是 PyIceberg，协议不兼容（[[polars-iceberg-oss-tables]]）无法靠换 Daft 绕开
2. **读 RW merge-on-read 表同样读出重复行**——Daft 不应用 equality deletes，而 RW iceberg upsert sink 恰以 merge-on-read 写入；项目手写的 [[iceberg-reader|IcebergDedupReader]]（按主键取最大 seq 去重）解决的正是这个问题，Daft 当前同样需要这层处理。湖读主链路实际主力是 pyiceberg + DuckDB，Polars 只在两端，Daft 插不进

**Daft 的真实进场时机**（未来）：

1. 读路径支持 V2 equality deletes 后 → `daft.read_iceberg()` 可替代手写 dedup reader
2. 数据量大到单机放不下 → Ray 分布式
3. 出现图像/嵌入管道 → 多模态算子

三条目前均不满足，定位保持"地图点位，不入栈"。另：RisingWave 的 CDC 流式 upsert 不可被 Daft 替代——Daft 是批处理引擎，无变更捕获能力，upsert 语义也不支持。

## 参见

- [[polars]] — 单机高性能对照物
- [[iceberg]] — Daft（经 PyIceberg）支持的表格式
- [[polars-iceberg-oss-tables]] — Polars 写 OSS Tables 的协议兼容性问题（Daft 同受影响）
- [[task-data-pipeline]] — 项目湖仓链路（RW CDC 不可替代的语境）
- [[lakehouse]] — 湖仓一体架构
- [[hnsw-index]] — Embedding 列的下游消费（向量检索）
