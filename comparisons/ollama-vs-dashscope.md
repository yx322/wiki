---
title: Ollama vs DashScope 嵌入方案对比
created: 2026-07-21
updated: 2026-07-21
type: comparison
tags: [comparison, dashscope, embedding, surrealdb]
sources: [raw/articles/SurrealDB向量集成方案分析.md]
confidence: high
---

# Ollama vs DashScope 嵌入方案对比

## 对比维度

| 维度 | Ollama (bge-m3) | DashScope (text-embedding-v4) | 双模并行 |
|------|-----------------|-------------------------------|---------|
| **基础设施** | 需部署，占显存/内存 | 零部署，仅依赖网络 | 两者兼顾 |
| **配置复杂度** | IP + 端口 + 模型名 | 仅 API Key | 维护两套配置 |
| **集成方式** | `fn::ollama::embed`（内置） | `fn::dashscope_embed`（自定义） | 动态路由 |
| **向量维度** | 1024 | 1024 | — |
| **性能瓶颈** | 硬件算力 (GPU/CPU) | 网络延迟 + API QPS | 互补 |
| **语义质量** | 中等 | 更高 | 可 A/B 测试 |
| **成本** | 硬件成本 | API 调用费 | 两者皆有 |

## 适用场景

### Ollama
- 数据敏感、内网环境
- 高并发、低延迟要求
- 不想依赖外部 API

### DashScope
- 快速启动、不想管服务器
- 追求更高语义理解质量
- 数据量适中、并发不高

### 双模并行
- A/B 测试对比效果
- 容灾降级（Ollama 挂了自动切 DashScope）
- 日常走本地、疑难走云端

## 双模并行架构

```
数据存储：vec_ollama + vec_dashscope（两个字段）
搜索路由：读 config 表的 search_strategy 决定用哪个字段
降级策略：Ollama 结果为空 → 自动用 DashScope 再搜一次
```

## 结论

项目最终选择了纯 DashScope 方案（[[embedding-migration]]），原因：
1. 不需要维护本地模型服务
2. 语义质量更高
3. 并发量不需要本地 GPU 支撑

## 参见
- [[dashscope]] — DashScope 详情
- [[embedding-migration]] — 迁移指南
- [[surrealdb]] — 数据库

^[raw/articles/SurrealDB向量集成方案分析.md]
