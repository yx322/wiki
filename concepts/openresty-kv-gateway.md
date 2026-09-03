---
title: OpenResty + KV 网关
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [embedded-kv, architecture, performance, redis]
sources: [raw/articles/kv-storage-engine.md]
confidence: high
---

# OpenResty + KV 网关

OpenResty 处理 HTTP 仍是最优解（LuaJIT + cosocket 非阻塞 I/O）。变化是将 Redis 替换为本地 Rust Sidecar + [[fjall]]，通过 Unix Socket 通信——网络 RTT 降为进程间 μs 级调用。

## 架构

```
[Client] → [OpenResty (Lua, HTTP 层)] → [KV Sidecar (Rust, Unix Socket)]
                                            ↓
                                        [Fjall LSM-Tree]
```

Sidecar 协议极简：4 字节长度前缀的二进制帧 `[len:4][op:1][key_len:4][key:N][value_len:4][value:M]`，四个操作 Get / Put / Delete / PrefixScan。

## 复合键编排

| 域 | Key 编码 | Value | 查询模式 |
|----|---------|-------|---------|
| **限流** | `rl:{client_id}:{fixed_window_ts}` | `count` (i64) | 点查当前窗口 |
| **响应缓存** | `cache:{method}:{path_hash}:{etag_hash}` | `{status, headers, body}` | 点查 + 前缀扫描批量失效 |
| **会话** | `sess:{user_id}:{session_id}:{field}` | 字段值 | 前缀扫描加载全部字段 |
| **熔断** | `cb:{upstream_id}` | `{state, failure_count}` | 纯点查 |
| **动态路由** | `route:{method}:{priority_zp}:{path_pattern}` | `{upset, timeout}` | 前缀按 method 扫描 |

## 关键设计点

### 限流：固定窗口 vs 滑动窗口

固定窗口 `rl:192.168.1.1:1722500000`（当前 10 分钟窗口 Unix 时间戳）点查计数，超限拒绝。固定窗口在网关场景够用且快一个数量级。

### 缓存：前缀扫描批量失效

传统 Redis 用 `SCAN` 游标迭代做路径前缀失效（阻塞单线程）。KV 的前缀扫描是 LSM-Tree 原生能力——迭代器 `Seek("cache:GET:")` 直接定位，批量 Delete 写 Tombstone，不阻塞前台请求。

### 路由表：有序一次加载

`route:GET:0010:/api/v1/*` 中 priority 用 4 位零填充保证字典序 = 数值序。OpenResty 启动时一次前缀扫描加载全量路由表到 nginx 共享字典，运行时零 KV 访问。

## 与旧 Redis 网关设计的区别

| 维度 | 旧设计（Redis） | 新设计（KV Sidecar） |
|------|----------------|---------------------|
| 限流 + 缓存写入 | 两次网络请求，无原子性 | 原子 Batch 同时提交 |
| SCAN 批量失效 | 阻塞单线程事件循环 | LSM-Tree 后台 compaction |
| 重启恢复 | RDB/AOF，分钟级 | WAL + SSTable，秒级 |
| 并发 | 单线程串行 | Tokio 多线程，多核并行 |
| 内存占用 | 全量驻留 RAM | 热数据 MemTable，冷数据 SSD |

## 关联页面

- [[fjall]] — 底层存储引擎
- [[composite-key-encoding]] — 复合键编码
- [[design-patterns]] — Bitmap 前置拦截模式
- [[embedded-kv-vs-redis]] — 与 Redis 的全面对比
