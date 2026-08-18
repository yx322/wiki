# 网络协议深度解析：从 HTTP 到 SSE、WebSocket 与 HTTP/3
> 本文档整理自一次关于网络协议的技术对话，涵盖 HTTP 基础、SSE、WebSocket、TCP 关系以及 HTTP/3 底层原理。
## 目录
1. [布隆过滤器（Bloom Filter）](#1-布隆过滤器bloom-filter)
2. [SSE（Server-Sent Events）从 HTTP 协议角度分析](#2-sseserver-sent-events从-http-协议角度分析)
3. [WebSocket 从 HTTP 协议角度分析](#3-websocket-从-http-协议角度分析)
4. [HTTP 协议简介](#4-http-协议简介)
5. [HTTP 与 TCP 的关系](#5-http-与-tcp-的关系)
6. [HTTP/3 为什么基于 UDP](#6-http3-为什么基于-udp)
---
## 1. 布隆过滤器（Bloom Filter）
### 1.1 什么是布隆过滤器？
**布隆过滤器是一种空间效率极高的概率型数据结构**，用于判断**一个元素是否可能存在于一个集合中**。
它的回答只有两种：
- **"不在"** —— 确定不在。
- **"可能在"** —— 有一定概率误判（假阳性）。
也就是说，它**允许误判（把不在的说成在），但绝不错判（把在的说成不在）**。
### 1.2 为什么需要它？
传统数据结构（如哈希表、数组）判断成员时，需要存储元素本身，空间开销大。布隆过滤器用极小的内存（比特位）就能表示海量数据，常用于**大量数据、内存受限、允许少量误判**的场景。
### 1.3 工作原理
布隆过滤器由两部分组成：
- **一个位数组（bit array）**，长度 m，初始所有位为 0。
- **k 个独立的哈希函数**，每个函数将元素映射到 [0, m-1] 的一个位置。
**添加元素：**
1. 对元素计算 k 个哈希值。
2. 将位数组中对应的 k 个位置全部设为 1。
**查询元素：**
1. 对元素计算 k 个哈希值。
2. 检查这 k 个位置是否**全部为 1**。
- 如果有一个为 0 → **一定不存在**。
- 如果全部为 1 → **可能存在**（因为可能是其他元素设置的）。
### 1.4 优缺点
**优点：**
- 空间极小（比存储原始数据小几个数量级）。
- 查询/插入时间复杂度 O(k)，k 为常数，非常快。
- 数据本身不存储，有一定安全性。
**缺点：**
- 有假阳性（误判），无法删除元素（除非用计数布隆过滤器）。
- 一旦构造好，很难动态扩容（需重建）。
- 需要合理选择 m 和 k 来平衡误判率。
### 1.5 误判率与参数选择
误判率大约为：
```
(1 - e^(-kn/m))^k
```
- m 越大，误判率越低。
- k 最优值约为：`k = (m/n) * ln 2`，其中 n 是预估元素数量。
通常设计时先定 n 和可接受误判率 p，再反推 m 和 k。
### 1.6 常见应用场景
| 场景 | 说明 |
|------|------|
| **缓存穿透防护** | 查询缓存前先用布隆过滤器判断 key 是否可能存在，避免大量请求打到数据库 |
| **黑名单/白名单过滤** | 如垃圾邮件地址、恶意 URL 过滤 |
| **数据库查询加速** | 如 HBase、Cassandra 用布隆过滤器快速判断某行是否存在 |
| **爬虫 URL 去重** | 海量 URL 是否已爬取 |
| **分布式系统** | 如 Google Bigtable、Apache Kafka 等内部使用 |
### 1.7 变种（扩展）
- **计数布隆过滤器（Counting Bloom Filter）**：位变成计数器，支持删除。
- **可扩展布隆过滤器（Scalable Bloom Filter）**：动态增加容量。
- **布谷鸟过滤器（Cuckoo Filter）**：支持删除且误判率更低，但实现更复杂。
---
## 2. SSE（Server-Sent Events）从 HTTP 协议角度分析
### 2.1 SSE 本质上是"标准 HTTP 请求"的升级
SSE 并不是什么新协议，它就是**一个标准的 HTTP GET 请求**，只是在请求头和响应头上做了特殊约定，让服务器可以**分块持续推送数据**。
### 2.2 客户端请求（Request）
```
GET /events HTTP/1.1
Host: api.example.com
Accept: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```
关键点：
- `Accept: text/event-stream` —— 告诉服务器客户端期望接收 SSE 流。
- `Cache-Control: no-cache` —— 防止代理或浏览器缓存这个请求。
- `Connection: keep-alive` —— 明确要求复用 TCP 连接。
### 2.3 服务器响应（Response）
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked
```
| 响应头 | 含义 |
|--------|------|
| `Content-Type: text/event-stream` | MIME 类型固定，浏览器识别后进入 SSE 模式 |
| `Cache-Control: no-cache` | 禁止中间节点缓存，保证实时性 |
| `Connection: keep-alive` | 保持 TCP 连接不关闭 |
| `Transfer-Encoding: chunked` | **核心**，分块传输，长度未知，持续发送 |
### 2.4 数据格式
SSE 的数据格式是**纯文本**，按行分割：
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
协议规则：
- 每条消息以 `data:` 开头。
- 多个 `data:` 行合并成一个消息。
- 消息以空行（`\n\n`）结束。
- 可选字段：`event:`、`id:`、`retry:`
- 以 `:` 开头的行是注释，常用于心跳。
### 2.5 重连与断线恢复
- 客户端断开后，浏览器自动重新发起 HTTP 请求。
- 如果服务器发送过 `id:`，浏览器重连时携带 `Last-Event-ID` 头。
- 服务器可根据 ID 从断点继续推送，实现断点续传。
### 2.6 SSE vs WebSocket（HTTP 视角）
| 对比项 | SSE | WebSocket |
|--------|-----|-----------|
| 协议基础 | 纯 HTTP（text/event-stream + chunked） | 握手用 HTTP，数据用独立 WS 协议 |
| 通信方向 | 单向（服务器→客户端） | 全双工（双向） |
| 数据格式 | 纯文本（固定格式） | 二进制或文本（自定义） |
| 自动重连 | 内置支持 | 需手动实现 |
| 防火墙友好 | 是 | 握手是 HTTP，后续非 HTTP 流量 |
---
## 3. WebSocket 从 HTTP 协议角度分析
### 3.1 WebSocket 的"双重身份"
- **握手阶段（Handshake）**：使用 **HTTP/1.1 协议**，携带特殊头部，请求协议升级。
- **数据传输阶段**：**不再是 HTTP 协议**，而是独立的 **WebSocket 帧协议（RFC 6455）**。
### 3.2 握手请求（Client → Server）
```
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
Sec-WebSocket-Extensions: permessage-deflate
```
| 请求头 | 含义 |
|--------|------|
| `Upgrade: websocket` | 核心！将 HTTP 协议升级为 WebSocket |
| `Connection: Upgrade` | 配合 Upgrade，表示升级请求 |
| `Sec-WebSocket-Key` | 16 字节随机数（Base64），用于握手验证 |
| `Sec-WebSocket-Version` | 固定为 `13`（RFC 6455 标准） |
### 3.3 握手响应（Server → Client）
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Extensions: permessage-deflate
```
| 响应头 | 含义 |
|--------|------|
| `101 Switching Protocols` | **状态码 101**，表示协议切换成功 |
| `Upgrade: websocket` | 确认切换到 WebSocket |
| `Sec-WebSocket-Accept` | 对客户端 Key 的 SHA-1 哈希（盐值固定） |
计算规则：
```
accept = base64(sha1(sec-websocket-key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
```
### 3.4 握手完成后
一旦服务器返回 101 且验证通过，**这条 TCP 连接不再走 HTTP 协议**：
- HTTP 的请求/响应模型彻底作废。
- 后续数据按 **WebSocket 帧格式** 封装。
- 不再有 HTTP 头部、状态码、Content-Type。
### 3.5 WebSocket vs SSE（完整对比）
| 对比维度 | SSE | WebSocket |
|----------|-----|-----------|
| 协议基础 | 全程 HTTP/1.1 | 握手用 HTTP，数据用独立 WS 协议 |
| 状态码 | 200 OK（始终） | 101 Switching Protocols |
| 数据格式 | 纯文本（UTF-8） | 二进制或文本（自定义帧） |
| 通信方向 | 单向（服务器→客户端） | 全双工（双向） |
| HTTP 头部 | 每次数据仍是 HTTP body 的一部分 | 握手后无 HTTP 头部，开销为 0 |
| 自动重连 | 原生支持 | 需手动实现 |
| 二进制支持 | 否 | 是 |
| 协议开销 | 每个消息有 HTTP 分块开销 | 帧头部 2~14 字节，极小 |
---
## 4. HTTP 协议简介
### 4.1 HTTP 是什么？
**HTTP（HyperText Transfer Protocol，超文本传输协议）** 是应用层协议，基于 **TCP/IP**，用于 Web 浏览器和服务器之间的通信。
它是**无状态**的（每个请求独立），但通过 Cookie/Session 等方式可以模拟状态。
### 4.2 核心模型：请求-响应
HTTP 遵循 **一问一答** 模式：
- 客户端发起请求（Request）
- 服务器返回响应（Response）
### 4.3 HTTP 报文结构
**请求报文：**
```
请求行（Method + URI + 版本）
请求头（Headers）
空行（\r\n）
请求体（Body，可选）
```
示例：
```
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: Chrome
Accept: text/html
```
**响应报文：**
```
状态行（版本 + 状态码 + 状态描述）
响应头（Headers）
空行
响应体（Body）
```
示例：
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1024
<html>...</html>
```
### 4.4 核心要素
**请求方法（Method）：**
| 方法 | 含义 |
|------|------|
| GET | 获取资源 |
| POST | 提交数据（创建） |
| PUT | 更新资源（全量） |
| DELETE | 删除资源 |
| PATCH | 部分更新 |
| HEAD | 只获取响应头 |
| OPTIONS | 查询服务器支持的请求方法 |
**状态码（Status Code）：**
| 类别 | 含义 | 示例 |
|------|------|------|
| 1xx | 信息性 | 101 Switching Protocols |
| 2xx | 成功 | 200 OK, 204 No Content |
| 3xx | 重定向 | 301, 302, 304 |
| 4xx | 客户端错误 | 400, 401, 403, 404 |
| 5xx | 服务端错误 | 500, 502, 503 |
### 4.5 HTTP 版本演进
| 版本 | 特点 |
|------|------|
| **HTTP/0.9** | 只有 GET，无头，纯文本，已废弃 |
| **HTTP/1.0** | 引入状态码、头部、POST，但每个请求新建 TCP 连接 |
| **HTTP/1.1** | 持久连接（keep-alive）、管道化、分块传输，目前最广泛 |
| **HTTP/2** | 二进制帧、多路复用、头部压缩、服务器推送 |
| **HTTP/3** | 基于 UDP 的 QUIC 协议，解决队头阻塞 |
---
## 5. HTTP 与 TCP 的关系
### 5.1 一句话概括
> **TCP 是快递公司（负责把包裹安全送到），HTTP 是包裹上的发货单（写清楚里面是什么、给谁、怎么处理）。HTTP 本身不关心路怎么走，全交给 TCP 去搞定。**
### 5.2 网络分层模型
| TCP/IP 四层 | 代表协议 | 作用 |
|-------------|----------|------|
| **应用层** | HTTP、FTP、WebSocket | 规定数据格式、语义 |
| **传输层** | **TCP、UDP** | 提供端到端的可靠/不可靠传输 |
| **网络层** | IP | 寻址和路由 |
| **网络接口层** | 以太网、WiFi | 物理介质传输 |
**HTTP 在应用层，TCP 在传输层。应用层协议必须通过传输层协议来实际发送数据。**
### 5.3 HTTP 依赖 TCP 的具体体现
| 特性 | TCP 为 HTTP 提供的能力 |
|------|------------------------|
| **可靠性** | 保证数据不丢失、不重复、顺序正确 |
| **连接性** | HTTP/1.1 的 keep-alive 复用 TCP 连接 |
| **流控制** | TCP 自动调节发送速度 |
| **分段传输** | HTTP 的 chunked 依赖 TCP 分段 |
### 5.4 为什么不直接基于 IP 或 UDP？
- 基于 **IP**：不可靠，包可能丢失、乱序，HTTP 需自己实现重传。
- 基于 **UDP**：也不可靠，适合实时场景（直播、游戏），不适合网页请求。
**HTTP 选择了 TCP，因为它提供了可靠性，让 HTTP 可以专注于业务逻辑。**
### 5.5 关键区别
| 对比维度 | HTTP | TCP |
|----------|------|-----|
| **层级** | 应用层（第7层） | 传输层（第4层） |
| **职责** | 定义数据格式、语义 | 定义如何建立连接、如何可靠传输 |
| **数据单位** | 报文（Message） | 数据段（Segment） |
| **可靠性** | 依赖 TCP 实现 | 自身保证可靠 |
| **可读性** | 文本协议 | 二进制协议 |
| **端口** | 默认 80/443 | 无固定端口 |
### 5.6 SSE/WebSocket 与 TCP
| 协议 | 和 TCP 的关系 |
|------|---------------|
| **HTTP** | 直接运行在 TCP 之上 |
| **SSE** | 本质是 HTTP，运行在 TCP 之上 |
| **WebSocket** | 握手走 HTTP（→TCP），握手后直接基于 TCP 裸连接 |
---
## 6. HTTP/3 为什么基于 UDP
### 6.1 一句话概括
> **HTTP/3 选择 UDP 不是因为它不可靠，而是因为 UDP 足够"薄"，让 Google 在它之上重新实现了一个"用户态 TCP"（QUIC），顺便把 TCP 几十年来改不了的问题（队头阻塞、握手慢、不支持连接迁移）一次性解决了。**
### 6.2 痛点：TCP 的"队头阻塞"
TCP 是**可靠且有序**的协议，如果传输过程中丢失了一个数据包，后续所有数据包都必须等待这个包被重传并收到，才能交给应用层。
```
发送方： [包1] [包2] [包3] [包4] [包5]
传输中： 包1到达，包2丢失，包3到达，包4到达，包5到达
接收方： 包1 到达 -> 等待包2... -> 包3、4、5虽然已到，但被堵在TCP缓冲区
```
**在 HTTP/2 多路复用场景下，任意一个流丢包，会阻塞整条连接上的所有流。**
### 6.3 为什么不能改进 TCP？
- TCP 在**内核态**实现，升级需要操作系统更新，周期极长。
- **中间设备**对 TCP 做了大量优化和硬编码，改动会导致失效。
- TCP 的"三次握手 + TLS 握手"延迟很高（至少 2-3 个 RTT）。
Google 选择了在**用户态基于 UDP**重新实现"类 TCP"协议——**QUIC**。
### 6.4 HTTP/3 核心架构
```
HTTP/1.1 和 HTTP/2：
+----------+
| HTTP     | 应用层
+----------+
| TLS      | 安全层
+----------+
| TCP      | 传输层（内核）
+----------+
| IP       | 网络层
+----------+

HTTP/3：
+----------+
| HTTP     | 应用层（语义完全不变）
+----------+
| QUIC     | 传输层（用户态，基于UDP）
+----------+
| UDP      | 传输层（内核，几乎无逻辑）
+----------+
| IP       | 网络层
+----------+
```
### 6.5 HTTP/3/QUIC 解决了哪些问题？
**1. 解决队头阻塞（流级别而非连接级别）**
```
一条 QUIC 连接上：
- 流A：加载 HTML 完成
- 流B：加载大图片（丢包，等待重传）-> 只有流B卡住
- 流C：加载 CSS 完成 继续传输
```
**2. 0-RTT 连接建立**
| 方式 | 耗时 |
|------|------|
| 传统 TCP+TLS 1.3 | 2 个 RTT |
| QUIC 首次连接 | 1 个 RTT |
| QUIC 恢复连接（0-RTT） | 0 个 RTT（第一个包直接携带数据） |
**3. 连接迁移（Connection Migration）**
- TCP 连接由 IP + 端口四元组标识，IP 变化则连接中断。
- QUIC 使用 64 位 **Connection ID**，与 IP/端口解耦。
- WiFi 切换到 5G 时连接不中断，应用无感知。
**4. 更灵活的拥塞控制**
- TCP 拥塞控制在内核，升级慢。
- QUIC 在用户态，可快速实验和部署新算法（如 BBR）。
**5. 减少头部开销**
| 协议 | 头部大小 |
|------|----------|
| TCP + TLS + HTTP/2 | ~100+ 字节 |
| QUIC + HTTP/3 | ~20-50 字节 |
### 6.6 挑战与代价
| 挑战 | 说明 |
|------|------|
| **UDP 被部分网络限制** | 某些企业防火墙、老旧 NAT 会阻断 UDP |
| **CPU 开销更高** | 用户态 QUIC 比内核态 TCP 消耗更多 CPU |
| **实现复杂度高** | 需重新实现拥塞控制、丢包检测等 |
| **生态迁移成本** | 服务器、CDN、浏览器、操作系统需全面支持 |
| **中间设备适配** | 某些 LB、代理对 UDP 支持不完善 |
目前 **Chrome、Firefox、Edge 都默认支持 HTTP/3**，主流 CDN（Cloudflare、Akamai）和大型网站（Google、Facebook、YouTube）已全面启用。
### 6.7 总结对比
| 对比维度 | HTTP/1.1 & HTTP/2 | HTTP/3 |
|----------|-------------------|--------|
| **传输层协议** | TCP | UDP + QUIC |
| **队头阻塞** | 有（TCP 连接级别） | 无（QUIC 流级别） |
| **连接建立** | 2-3 RTT | 0-1 RTT |
| **加密** | TLS（独立层） | 内置 TLS 1.3 |
| **连接迁移** | 不支持 | 支持（Connection ID） |
| **拥塞控制升级** | 需更新内核 | 用户态随应用升级 |
| **头部大小** | 较大 | 更小 |
---
## 结语
以上涵盖了从布隆过滤器、SSE、WebSocket 到 HTTP 基础、TCP 关系以及 HTTP/3 底层原理的完整技术脉络。这些协议共同构成了现代 Web 通信的基石，理解它们的层次关系和设计取舍，对于做网络编程、性能优化和架构设计都至关重要。
---
*整理日期：2026年8月*
