---
title: 纯 KV 四大设计模式
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, lsm-tree, performance, architecture]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# 纯 KV 四大设计模式

在纯 KV 世界里，算法不再是游离在数据库外面的胶水代码——Key 编码格式本身就是索引层、缓存层和隔离层。

## 模式一：Index-Only Scan（索引即数据）

二级索引的 Value 留空，entity_id 编码在 Key 末尾。查询时仅遍历 Key 序列即可获取所有匹配的主键 ID，无需回表读取 Value——零磁盘 I/O。

```
数据主表：
  d:{tenant}:{type}:{id}  →  [完整业务实体]

属性索引表：
  i:{tenant}:{type}:{attr}:{value}:{id}  →  ""  (Value 留空)

查询「状态为 active 的所有用户」：
  Scan("i:1:user:status:active:") → 遍历 Key 序列 → 提取末尾 id
  无需读取任何 Value，零回表
```

前缀扫描只触碰 LSM-Tree 的 MemTable + 索引层 SSTable（极小），不碰数据层的大 Value 文件。

## 模式二：Bitmap 前置拦截（热路径零 KV 调用）

网关高频检查"IP 是否在黑名单""Token 是否合法"。在应用层内存常驻 Roaring Bitmap：

```
读路径（零 KV 调用）：
  网关收到请求
    → bitmap.check(hash(ip))
    → 未命中 → 直接放行（QPS 百万级，纯 CPU 位判断）
    → 可能命中 → kv.get() 最终权威判定
```

99%+ 的正常请求完全在 CPU L1 Cache 内完成，不触发任何 KV 引擎调用。

**Bitmap vs 布隆过滤器**：Bitmap 支持精确删除，布隆过滤器只增不删。高频变更的黑名单用 Bitmap；只增不减的 Token 白名单用布隆过滤器。

## 模式三：应用层 MVCC（无原生 MVCC 引擎的时间旅行）

[[fjall]]/[[slatedb]] 不支持原生 MVCC。将版本号编排进 Key 骨架：

```
Key: data:agent:101:v:[u64::MAX - 1001]  →  记忆状态 v1001
Key: data:agent:101:v:[u64::MAX - 1002]  →  记忆状态 v1002（最新）
```

- 常规读取：`Scan("data:agent:101:v:").next()` → 补码反转后最新版本排最前
- 时间旅行：将前缀指针定位到目标版本号之后，正向扫描即得历史版本链

与 [[surrealkv]] 原生 MVCC 的区别：SurrealKV 内置 `tx.get_at(key, timestamp)` 直接查询历史版本，不需要应用层编码。

→ 详见 [[application-mvcc]]

## 模式四：WiscKey 键值分离（大 Value 场景的写放大解药）

Key 几十字节，Value 几 MB 时，传统 LSM-Tree Compaction 将 Key+Value 捆绑重写，写放大几十倍。

WiscKey 核心：Key 和 Value 物理分离——索引层只存小指针，大 Value 追加写入独立日志文件。

```
【内存 + SSTable 索引层】
 Key: "asset:mesh:uuid_abc" → [File_ID : Offset : Length]
                                    │
                                    ▼ 磁盘随机点查
【独立 Value Log（顺序追加）】
 offset_3402 → [大体积二进制资产]
```

- Compaction 只搬动小指针（几字节），写放大从几十倍降为接近 1
- 读取：索引定位指针后，一次磁盘随机点查（NVMe ~10μs）

[[fjall]] 3.0 原生支持 WiscKey，[[surrealkv]] 通过 Blob Log 实现同等效果。

→ 详见 [[wisckey-separation]]

## 关联页面

- [[fjall]] — 底层存储引擎
- [[composite-key-encoding]] — Key 编码基础
- [[application-mvcc]] — 应用层 MVCC 详解
- [[wisckey-separation]] — WiscKey 详解
- [[embedded-kv-vs-redis]] — 与 Redis 的对比
