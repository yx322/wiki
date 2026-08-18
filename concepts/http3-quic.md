---
title: HTTP/3 与 QUIC
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [network, http3, quic, udp, tcp]
sources: [raw/articles/network-protocols-deep-dive.md]
confidence: high
---

# HTTP/3 与 QUIC

**核心论点**：HTTP/3 选择 UDP 不是因为 UDP 不可靠，而是因为 UDP 足够"薄"——让 Google 在用户态重新实现了一个"类 TCP"协议（QUIC），一次性解决 TCP 几十年来改不了的问题。

## TCP 的痛点：队头阻塞

TCP 可靠且有序——丢一个包，后续所有包都要等重传：

```
发送方：[包1] [包2] [包3] [包4] [包5]
接收方：包1 ✓ → 等包2... → 包3、4、5 已到但被堵在 TCP 缓冲区
```

HTTP/2 多路复用下，**任意一个流丢包 → 阻塞整条连接的所有流**。

## 为什么不能改 TCP？

- TCP 在**内核态**实现，升级需要操作系统更新
- 中间设备对 TCP 做了大量硬编码优化
- TCP 三次握手 + TLS 握手 = 至少 2-3 个 RTT

## 架构对比

```
HTTP/1.1 和 HTTP/2：       HTTP/3：
┌─────────┐               ┌─────────┐
│  HTTP   │               │  HTTP   │ 语义不变
├─────────┤               ├─────────┤
│  TLS    │               │  QUIC   │ 用户态传输
├─────────┤               ├─────────┤
│  TCP    │ 内核态         │  UDP    │ 内核态（几乎无逻辑）
├─────────┤               ├─────────┤
│  IP     │               │  IP     │
└─────────┘               └─────────┘
```

## QUIC 解决的五大问题

| 问题 | TCP/HTTP/2 | QUIC/HTTP/3 |
|------|-----------|------------|
| **队头阻塞** | 连接级别阻塞 | 流级别阻塞（流B卡住不影响流C）|
| **连接建立** | 2-3 RTT | 0-1 RTT（0-RTT 恢复连接）|
| **加密** | TLS 独立层 | 内置 TLS 1.3 |
| **连接迁移** | IP 变化 = 断连 | Connection ID 解耦 IP/端口 |
| **拥塞控制** | 内核态，升级慢 | 用户态，快速实验（如 BBR）|

## 0-RTT 连接建立

| 方式 | 耗时 |
|------|------|
| TCP + TLS 1.3 | 2 个 RTT |
| QUIC 首次连接 | 1 个 RTT |
| QUIC 恢复连接 | 0 个 RTT（第一个包直接携带数据）|

## 连接迁移

- TCP：由 IP + 端口四元组标识 → IP 变化 = 断连
- QUIC：使用 64 位 **Connection ID** → WiFi 切 5G 连接不中断

## 头部开销

| 协议 | 头部大小 |
|------|---------|
| TCP + TLS + HTTP/2 | ~100+ 字节 |
| QUIC + HTTP/3 | ~20-50 字节 |

## 挑战与代价

| 挑战 | 说明 |
|------|------|
| UDP 被部分网络限制 | 企业防火墙/老旧 NAT 可能阻断 |
| CPU 开销更高 | 用户态 QUIC > 内核态 TCP |
| 实现复杂度 | 需重新实现拥塞控制、丢包检测 |
| 生态迁移 | 服务器/CDN/浏览器/OS 全面支持 |

目前 Chrome/Firefox/Edge 默认支持 HTTP/3，主流 CDN（Cloudflare、Akamai）和大型网站已全面启用。

## 参见
- [[sse-server-sent-events]] — SSE 协议
- [[websocket]] — WebSocket 协议

^[raw/articles/network-protocols-deep-dive.md]
