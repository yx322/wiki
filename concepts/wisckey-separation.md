---
title: WiscKey 键值分离
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, lsm-tree, performance, architecture]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# WiscKey 键值分离

大 Value 场景（Key 几十字节，Value 几 MB）的写放大解药。

## 问题：传统 LSM-Tree 的写放大

典型场景：对话历史、3D 资产二进制、多模态特征向量、日志原始载荷。

传统 LSM-Tree 在 Compaction 时将 Key+Value 捆绑重写，写放大几十倍。

## 解决方案：物理分离

WiscKey 核心：Key 和 Value 物理分离——索引层只存小指针，大 Value 追加写入独立日志文件。

```
【内存 + SSTable 索引层】
 Key: "asset:mesh:uuid_abc" → [File_ID : Offset : Length]
                                    │
                                    ▼ 磁盘随机点查
【独立 Value Log（顺序追加）】
 offset_3402 → [大体积二进制资产]
```

## 物理效果

- **Compaction**：只搬动小指针 Key（几字节），不碰大 Value 文件。写放大从几十倍降为接近 1
- **读取**：索引定位指针后，一次磁盘随机点查（NVMe 上 ~10μs）抓取大 Value
- **适用场景**：任何 Key 小 Value 大的模式——Agent 对话历史、3D 资产、多模态特征、日志载荷

## 各引擎实现

| 引擎 | 实现方式 |
|------|---------|
| [[fjall]] 3.0 | 原生支持 WiscKey（KV 分离） |
| [[surrealkv]] | Blob Log 大对象分离 |
| [[slatedb]] | 依赖 S3 的 Range Get 读取大 Value |

## 关联页面

- [[fjall]] — 底层存储引擎
- [[design-patterns]] — 四大设计模式之一
- [[composite-key-encoding]] — Key 编码基础
