---
title: SSE（Server-Sent Events）
created: 2026-08-18
updated: 2026-08-18
type: concept
tags: [network, sse, http, streaming]
sources: [raw/articles/network-protocols-deep-dive.md]
confidence: high
---

# SSE（Server-Sent Events）

SSE 不是一个新协议，它是**标准 HTTP GET 请求的升级**——在请求头和响应头上做特殊约定，让服务器可以分块持续推送数据。

## HTTP 层面的实现

### 客户端请求

```
GET /events HTTP/1.1
Host: api.example.com
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

### 服务器响应

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked
```

| 响应头 | 含义 |
|--------|------|
| `Content-Type: text/event-stream` | MIME 类型固定，浏览器进入 SSE 模式 |
| `Cache-Control: no-cache` | 禁止中间节点缓存，保证实时性 |
| `Transfer-Encoding: chunked` | **核心**：分块传输，长度未知，持续发送 |

## 数据格式

纯文本，按行分割：

```
data: 这是第一条消息\n\n
data: 这是第二条消息\n
data: 分两行发送的消息\n\n
event: custom\n
data: 自定义事件消息\n\n
id: 12345\n
data: 带ID的消息\n\n
: 这是注释（心跳）\n\n
```

- 每条消息以 `data:` 开头
- 消息以空行（`\n\n`）结束
- 可选字段：`event:`、`id:`、`retry:`
- `:` 开头的行是注释，常用于心跳

## 重连与断线恢复

- 客户端断开后，浏览器自动重新发起 HTTP 请求
- 服务器发送过 `id:` → 浏览器重连时携带 `Last-Event-ID` 头
- 服务器根据 ID 从断点续传

## 与 WebSocket 对比

| 对比项 | SSE | WebSocket |
|--------|-----|-----------|
| 协议基础 | 全程 HTTP/1.1 | 握手用 HTTP，数据用独立 WS 协议 |
| 状态码 | 200 OK | 101 Switching Protocols |
| 通信方向 | 单向（服务器→客户端）| 全双工（双向）|
| 数据格式 | 纯文本 | 二进制或文本 |
| 自动重连 | 内置支持 | 需手动实现 |
| 二进制支持 | ❌ | ✅ |
| 协议开销 | HTTP 分块开销 | 帧头 2~14 字节 |

## 与 skillforge 的关系

skillforge 的 `api_server.py` 同时支持 SSE（`POST /v1/chat/completions`）和 WebSocket（`/ws/chat`）两种流式输出方式。

## 参见
- [[websocket]] — WebSocket 协议
- [[http3-quic]] — HTTP/3 与 QUIC

^[raw/articles/network-protocols-deep-dive.md]
