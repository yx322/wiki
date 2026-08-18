---
title: 布隆过滤器（Bloom Filter）
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [data-structure, probabilistic, cache, filter]
sources: [raw/articles/network-protocols-deep-dive.md]
confidence: high
---

# 布隆过滤器（Bloom Filter）

**空间效率极高的概率型数据结构**——判断一个元素是否可能存在于集合中。

## 核心特性

回答只有两种：
- **"不在"** → 确定不在
- **"可能在"** → 有一定概率误判（假阳性）

允许误判（把不在的说成在），绝不错判（把在的说成不在）。

## 工作原理

由两部分组成：
- **位数组**（bit array），长度 m，初始全 0
- **k 个独立哈希函数**，映射到 [0, m-1]

**添加**：对元素算 k 个哈希值 → 对应位置设为 1

**查询**：算 k 个哈希值 → 检查是否全部为 1
- 有一个为 0 → **一定不存在**
- 全部为 1 → **可能存在**

## 误判率

```
(1 - e^(-kn/m))^k
```

- m 越大，误判率越低
- k 最优值 ≈ `(m/n) * ln 2`
- 设计时先定 n 和可接受误判率 p，再反推 m 和 k

## 优缺点

| 优点 | 缺点 |
|------|------|
| 空间极小 | 有假阳性 |
| 查询 O(k)，极快 | 无法删除（除非用计数布隆过滤器）|
| 不存储原始数据 | 难以动态扩容 |

## 应用场景

| 场景 | 说明 |
|------|------|
| **缓存穿透防护** | 先用布隆过滤器判断 key 是否存在，避免请求打穿 DB |
| **黑名单/白名单** | 垃圾邮件、恶意 URL 过滤 |
| **数据库查询加速** | HBase、Cassandra 判断某行是否存在 |
| **爬虫 URL 去重** | 海量 URL 是否已爬取 |

## 与 KV 存储的关系

在 [[design-patterns]] 的 Bitmap 前置拦截模式中，布隆过滤器用于网关热路径：

```
写路径：kv.put("blacklist:ip:1.1.1.1", "") + bloom.add("1.1.1.1")
读路径：bloom.check(ip) → 未命中 → 直接放行（零 KV 调用）
                        → 可能命中 → kv.get() 最终确认
```

**布隆过滤器 vs Bitmap**：Bitmap 支持精确删除，布隆过滤器只增不删。高频变更的黑名单用 Bitmap，只增不减的 Token 白名单用布隆过滤器更省内存。

## 变种

- **计数布隆过滤器**：位变成计数器，支持删除
- **可扩展布隆过滤器**：动态增加容量
- **布谷鸟过滤器**：支持删除且误判率更低

## 参见
- [[design-patterns]] — KV 设计模式（Bitmap 前置拦截）
- [[composite-key-encoding]] — 复合键编码

^[raw/articles/network-protocols-deep-dive.md]
