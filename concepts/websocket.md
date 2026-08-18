---
title: WebSocket 协议
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [network, websocket, http, protocol]
sources: [raw/articles/network-protocols-deep-dive.md]
confidence: high
---

# WebSocket 协议

WebSocket 具有"双重身份"：握手阶段使用 HTTP/1.1，数据传输阶段切换为独立的 WebSocket 帧协议（RFC 6455）。

## 握手请求（Client → Server）

```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

| 请求头 | 含义 |
|--------|------|
| `Upgrade: websocket` | 将 HTTP 协议升级为 WebSocket |
| `Connection: Upgrade` | 配合 Upgrade |
| `Sec-WebSocket-Key` | 16 字节随机数（Base64），握手验证 |
| `Sec-WebSocket-Version` | 固定为 13（RFC 6455）|

## 握手响应（Server → Client）

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

| 响应头 | 含义 |
|--------|------|
| `101 Switching Protocols` | 协议切换成功 |
| `Sec-WebSocket-Accept` | Key 的 SHA-1 哈希验证 |

计算规则：
```
accept = base64(sha1(sec-websocket-key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```

## 握手后

101 验证通过后，**TCP 连接不再走 HTTP 协议**：
- HTTP 请求/响应模型作废
- 数据按 WebSocket 帧格式封装
- 不再有 HTTP 头部、状态码、Content-Type

## 与 SSE 完整对比

| 对比维度 | SSE | WebSocket |
|----------|-----|-----------|
| 协议基础 | 全程 HTTP/1.1 | 握手用 HTTP，数据用独立 WS 协议 |
| 通信方向 | 单向（服务器→客户端）| 全双工（双向）|
| 数据格式 | 纯文本（UTF-8）| 二进制或文本（自定义帧）|
| HTTP 头部 | 数据是 HTTP body | 握手后无 HTTP 头部 |
| 自动重连 | 原生支持 | 需手动实现 |
| 协议开销 | HTTP 分块开销 | 帧头 2~14 字节，极小 |

## 参见
- [[sse-server-sent-events]] — SSE 协议
- [[http3-quic]] — HTTP/3 与 QUIC

^[raw/articles/network-protocols-deep-dive.md]
