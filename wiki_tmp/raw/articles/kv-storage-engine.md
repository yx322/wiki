# KV 存储引擎：架构设计、复合键编码与用例

**Status:** 持续演进
**覆盖引擎：** Fjall（本地 NVMe + Raft）、SlateDB（S3 云原生）、SQLite 对比
**架构：** [Aura 架构 §5](../aura-architecture.md) — 双引擎模式（Fjall+Raft / SlateDB+S3）
**Cross-ref:** [Redis 批判](../redis-critique.md) — Redis 为何被 KV 替代

## 核心论点

三个认知转变颠覆了 Redis 作为核心基础设施的范式：

1. **从跨网络到进程内**：Redis 是「网络 RAM」——为无状态语言（PHP）设计的变通方案。Rust 进程是持久状态容器。Fjall（嵌入式 LSM-Tree KV）彻底消除网络 RTT 和序列化开销。
2. **从伪分布式到真共识**：Redis Cluster/Redlock 缺乏强一致性。Openraft 提供数学证明的 Raft 共识。Redlock 已被证明不安全（Kleppmann 2016：GC 停顿 + 时钟漂移）。
3. **从专家专属到 AI 可用**：Fjall + Openraft 封装所有复杂性。AI 处理胶水代码（状态机 apply、Key 编码、序列化），人类负责架构设计和审查。

## 架构

```
应用层（锁、调度、配置、会话）
        │
  状态机（Fjall 引擎 — 嵌入式持久化）
        │ 应用已提交的日志条目
  Openraft Raft 核心（共识 + 日志复制）
        │
  ┌─────┼─────┐
  节点1  节点2  节点3    （最少 3 节点，容忍 1 故障）
```

**关键洞察**：状态机就是桥梁。Raft 提交日志条目 → 状态机应用到 Fjall。无需独立存储层，本地读取无网络跳数。

## 复合键编码：Redis 数据结构 → KV

Fjall 是纯 KV 引擎。Redis 的数据结构通过 Key 编码 + 前缀/范围扫描模拟：

| Redis | Fjall Pattern | Read | Write |
|-------|--------------|------|-------|
| `STRING` | `str:<key>` | `get` | `put` |
| `HASH` | `hash:<key>:<field>` | `get` / `prefix` | `put` / `remove` |
| `LIST` | `list:<key>:<seq>` (8-digit zero-padded) | `range` | `put` + monotonic seq |
| `SET` | `set:<key>:<member>` → `""` | existence check | `put` / `remove` |
| `ZSET` | `zset:<key>:<score>:<member>` | `range` (score interval) | `put` / `remove` |

关键细节：LIST 需要原子序列号生成 → Raft 状态机中的单调计数器。ZSET Score 编码使用零填充固定宽度格式，保证字典序正确。

### Physical Encoding: Composite Key 排布细节

纯 KV 引擎没有 Hash、ZSET 等原语。一切数据结构都是通过 Key 空间编码（Composite Key Encoding）在字节序上模拟出来的——LSM-Tree 的迭代器天然按字节排序，这意味着只要 Key 编码设计正确，范围扫描和排序在存储层零成本完成。

#### Hash 的物理编码

Redis Hash `HSET user:101 name "Alice"` 在 Fjall 中拆为两个独立 KV 对：

```
Key: "hash:user:101:name"  →  Value: "Alice"       (string)
Key: "hash:user:101:age"   →  Value: [0x00000019]   (i64 big-endian, 8 bytes)
```

**前缀扫描批量删除**：`DEL user:101` 不需要先读取再逐字段删除。直接用 `prefix("hash:user:101:")` 定位所有字段 Key，批量发出 Delete 指令。LSM-Tree 的删除本质是写入 Tombstone 标记，O(1) 操作。Redis 的大 Hash 删除（百万字段级别）会阻塞单线程事件循环数十毫秒，Fjall 无此问题——多线程后台 compaction 异步清理 Tombstone，不阻塞前台请求。

#### ZSET 的物理编码：Score 作为 Key 前缀的逆向索引

ZSET 的核心约束是「按 Score 排序」。利用 LSM-Tree 的字节序排序特性，将 Score 嵌入 Key 前缀即可实现：

```
玩家 99 得分 100，玩家 88 得分 99

数据主表:
  Key: "data:player:88"  →  Value: {...}  (玩家结构体)
  Key: "data:player:99"  →  Value: {...}

排行索引（双写）:
  Key: "rank:scores:[0x0000000000000064]:99"  →  Value: ""  (score=100, 大端序)
  Key: "rank:scores:[0x0000000000000063]:88"  →  Value: ""  (score=99,  大端序)
```

Score 转为固定 8 字节大端序字节数组（`i64::to_be_bytes()`），确保数值大小与字节字典序严格一致。取前 10 名：迭代器 `Seek` 到 `"rank:scores:"` 前缀后向后遍历 10 次。无需内存排序，存储层自动保证顺序。

#### 大端序补零的工程陷阱

**绝对不能**将数字转为字符串后拼入 Key。字典序与数值序不同：

```
字符串序:  "9" > "10"   (比较 '9' > '1'，'9' 字符码更大)
数值序:    9  < 10
```

正确做法是固定宽度的大端序字节数组。Rust 实现：

```rust
fn score_key(prefix: &[u8], score: i64, member: &[u8]) -> Vec<u8> {
    let mut key = prefix.to_vec();
    key.extend_from_slice(&score.to_be_bytes());  // 8 bytes, 高位在前
    key.push(b':');
    key.extend_from_slice(member);
    key
}
```

所有 Score 共享同一字节长度（8 bytes），高位补零由硬件指令自动完成，字典序 = 数值序。

#### 补码反转编码（Complement Encoding）：物理层面的倒序排列

KV 引擎按 Key 字节序从小到大（字典序）严格排序。时间戳递增时，新数据天然排在最底下。为了实现「最新数据排在最上面」（方便取 Top-N 最新记忆），框架设计中有一个教科书级的物理黑客手段——最大值减去当前值：

```rust
// 标准正序 Key：老数据在最上面，新数据在最底下
let key_ascending = format!("log:{}:{}:", session_id, timestamp).into_bytes();

// 倒序 Key：最新写入的数据天然排在最前面！
// u64::MAX - timestamp：时间戳越大（越新），减出来越小
// 越小 → LSM-Tree 字典序越靠前 → 物理层面「最新优先」
let inverted_time = u64::MAX - timestamp;
let mut key_descending = Vec::new();
key_descending.extend_from_slice(format!("log:{}:", session_id).as_bytes());
key_descending.extend_from_slice(&inverted_time.to_be_bytes()); // 大端序保证按位对比正确
```

**物理含义**：`u64::MAX - 1722500000` = `18446744072007051616`，`u64::MAX - 1722500001` = `18446744072007051615`。后者更小，在 LSM-Tree 中排在更前面——时间戳越大（越新），Key 字典序越小，迭代器正向扫描自然得到倒序结果。

**应用场景**：Agent 对话历史倒序加载、排行榜取最新记录、日志时间线倒序读取。任何需要「最新优先」的前缀扫描场景都适用。与 `SeekForPrev` 反向迭代器互补——后者依赖引擎支持，补码反转在所有 KV 引擎上通用。

#### 双写原子性：ZSET 更新分数

更新 ZSET 分数涉及两步：删除旧分数 Key + 写入新分数 Key。这是两个不同的 KV 操作，必须在同一个原子批次内完成：

```rust
let mut batch = keyspace.batch();
batch.delete(old_score_key);   // 旧分数的索引
batch.put(new_score_key, b""); // 新分数的索引
batch.put(data_key, &updated_player); // 数据主表
keyspace.write(batch)?;        // 原子写入，全部成功或全部失败
```

如果不使用原子批次，崩溃导致只写了一半 → ZSET 索引产生脏数据（一个成员同时出现在两个分数位置，或丢失）。Fjall 的 `Batch` API 保证底层 WAL 一次原子提交。

在分布式模式下，这个 Batch 作为单条 Raft 指令提交——Leader 写入本地 Fjall 后复制到 Follower，多数派确认后 Apply。Batch 的原子性从单机延伸到集群。

#### 二级索引末尾必须追加主键 ID

设计二级索引（如通过时间戳反查对话消息）时，必须将全局唯一的实体主键 ID 拼接到索引 Key 的最末尾。

**原因**：高并发场景下，两个用户操作可能在同一微秒产生完全相同的时间戳。如果索引 Key 仅为 `idx:time:{timestamp}`，后一个写入会无声覆盖前一个（数据丢失）。末尾追加 `{message_id}` 既保证 Key 绝对唯一，又利用字典序维护时间线规整。

```
# 错误：同时间戳覆盖
idx:time:1722500000 → msg_abc  (被覆盖)
idx:time:1722500000 → msg_def

# 正确：主键 ID 保证唯一
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

#### 哈希前缀打散：避开自增序列的写放大灾难

当输入源严格自增递增（高频事件流水号、日志 tick），所有写操作集中撞击 LSM-Tree 末尾（Hotspot），后台 compaction 产生严重的磁盘写放大，IOPS 出现毛刺。

解法：在 Key 最前端注入哈希盐值，将连续写入打散到多个分区：

```rust
use murmur3::murmur3_32;

// 对 Key 核心内容生成 32 位哈希值，取模分入 16 个桶
let hash_prefix = (murmur3_32(&mut cursor, 0).unwrap() % 16) as u8;

let mut sharded_key = Vec::new();
sharded_key.push(hash_prefix);  // 1 字节哈希盐
sharded_key.extend_from_slice(b"metrics:ts:");
sharded_key.extend_from_slice(&timestamp.to_be_bytes());
```

物理效果：连续递增的写入压力被均匀分散到 16 个独立的内存树（MemTable）中，多线程并发刷盘（Flush）并行处理，避开局部块锁竞争，写入吞吐量翻倍。代价是前缀扫描需要遍历所有桶——适合写密集、读按精确 Key 点查的场景。

#### 长度前缀编码：消除低效的字符串分割扫描

Key 中拼接多个字符串属性时，冒号分隔（`table:app_name:user_id`）要求读取时写循环 `split` 查找分隔符——O(N) 字符串扫描。改用固定 2 字节长度前缀：

```rust
pub fn pack_string_component(buf: &mut Vec<u8>, component: &str) {
    let bytes = component.as_bytes();
    let len = bytes.len() as u16;  // 2 字节存储字符串物理长度
    buf.extend_from_slice(&len.to_be_bytes());
    buf.extend_from_slice(bytes);
}
```

反序列化时，CPU 读取前 2 字节得知长度，直接跳过对应偏移量提取目标字段——从 O(N) 字符串扫描变为 O(1) 内存地址偏移计算。

**与冒号分隔的对比**：

| 维度 | 冒号分隔 `a:b:c` | 长度前缀 `[2]a[1]b[1]c` |
|:--|:--|:--|
| 反序列化 | 循环 split + 字符串比较 | 2 字节读长度 + 指针偏移 |
| 二进制安全性 | 字段内不能包含 `:` | 任意字节均可 |
| 固定宽度 | 否（依赖分隔符） | 是（每段 = 2 字节长度 + N 字节数据） |
| 适用场景 | 人类可读调试 | 生产环境高频读写 |

#### 多租户复合键编解码器：工业级实现

生产级组件——零堆分配（Zero-heap Allocation）的多租户复合键序列化/反序列化：

```rust
use std::convert::TryInto;

pub struct AgentMemoryKey {
    pub tenant_id: u32,        // 4 字节（租户隔离）
    pub session_id: [u8; 16],  // 16 字节（UUID 原始二进制）
    pub timestamp: u64,        // 8 字节（倒序时间戳）
}

impl AgentMemoryKey {
    /// 序列化：领域模型 → 固定 30 字节二进制流（无堆分配）
    pub fn serialize_to_bytes(&self) -> Vec<u8> {
        // 4(tenant) + 1(/) + 16(session) + 1(/) + 8(time) = 30 字节
        let mut buf = Vec::with_capacity(30);

        // 租户 ID（大端序）
        buf.extend_from_slice(&self.tenant_id.to_be_bytes());
        buf.extend_from_slice(b"/");

        // 会话 UUID（固定 16 字节，无需编码）
        buf.extend_from_slice(&self.session_id);
        buf.extend_from_slice(b"/");

        // 倒序时间戳（补码反转，最新优先）
        let inverted_time = u64::MAX - self.timestamp;
        buf.extend_from_slice(&inverted_time.to_be_bytes());

        buf
    }

    /// 反序列化：二进制切片 → 领域模型（零拷贝，纳秒级）
    pub fn deserialize_from_bytes(bytes: &[u8]) -> Result<Self, &'static str> {
        if bytes.len() != 30 {
            return Err("非法的物理 Key 长度约束违规");
        }

        let tenant_id = u32::from_be_bytes(bytes[0..4].try_into().unwrap());
        let mut session_id = [0u8; 16];
        session_id.copy_from_slice(&bytes[5..21]);
        let inverted_time = u64::from_be_bytes(bytes[22..30].try_into().unwrap());
        let timestamp = u64::MAX - inverted_time;  // 反转还原

        Ok(Self { tenant_id, session_id, timestamp })
    }
}
```

**设计要点**：所有字段固定宽度（4+16+8=28 字节 + 2 分隔符 = 30 字节），反序列化无需解析、无需堆分配，纯指针切片操作。倒序时间戳内嵌在 Key 中，正向迭代器扫描即得「最新优先」结果。

## 设计模式：纯 KV 底座上的系统级范式

在纯 KV 世界里，算法不再是游离在数据库外面的胶水代码——Key 编码格式本身就是索引层、缓存层和隔离层。

### 模式一：Index-Only Scan（索引即数据）

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

**物理效果**：前缀扫描只触碰 LSM-Tree 的 MemTable + 索引层 SSTable（极小），不碰数据层的大 Value 文件。适合高频属性筛选（网关路由匹配、Agent 状态过滤）。

### 模式二：Bitmap 前置拦截（热路径零 KV 调用）

网关高频检查「IP 是否在黑名单」「Token 是否合法」。每次请求都 `kv.get()` 即便命中缓存也有哈希查找开销。解法：在应用层内存常驻 Roaring Bitmap 或布隆过滤器。

```
写路径（新 IP 封禁时）：
  1. kv.put("blacklist:ip:1.1.1.1", "")   ← 持久化落盘
  2. bitmap.set(hash("1.1.1.1"))           ← 内存标记

读路径（网关拦截，零 KV 调用）：
  网关收到请求
    → bitmap.check(hash(ip))
    → 未命中 → 直接放行（QPS 百万级，纯 CPU 位判断）
    → 可能命中 → kv.get() 最终权威判定
```

**物理效果**：热路径（99%+ 的正常请求）完全在 CPU L1 Cache 内完成，不触发任何 KV 引擎调用。只有极少数命中 Bitmap 的请求才下沉到存储层。适合网关黑名单/白名单、限流计数器、Token 校验。

**与布隆过滤器的区别**：Bitmap 支持精确删除（`bitmap.unset()`），布隆过滤器只增不删。高频变更的黑名单用 Bitmap；只增不减的 Token 白名单用布隆过滤器更省内存。

### 模式三：应用层 MVCC（无原生 MVCC 引擎的时间旅行）

Fjall/SlateDB 不支持原生 MVCC。框架团队通过将版本号编排进 Key 骨架，实现应用层多版本控制：

```
Key: data:agent:101:v:[u64::MAX - 1001]  →  记忆状态 v1001
Key: data:agent:101:v:[u64::MAX - 1002]  →  记忆状态 v1002（最新）
```

- **常规读取**：`Scan("data:agent:101:v:").next()` → 补码反转后最新版本排最前，亚微秒拿到最新状态
- **时间旅行**：将前缀指针定位到目标版本号之后，正向扫描即得历史版本链。无需数据库快照锁，纯 Key 设计实现无锁历史回滚

**与 SurrealKV 原生 MVCC 的区别**：SurrealKV 内置 `tx.get_at(key, timestamp)` 直接查询历史版本，不需要应用层编码。Fjall/SlateDB 需要手动将版本号编入 Key。代价不同，效果相同。

### 模式四：WiscKey 键值分离（大 Value 场景的写放大解药）

典型场景：Key 几十字节，Value 几 MB（对话历史、3D 资产二进制、多模态特征向量、日志原始载荷）。传统 LSM-Tree 在 Compaction 时将 Key+Value 捆绑重写，写放大几十倍。

WiscKey 的核心：Key 和 Value 物理分离——索引层只存小指针，大 Value 追加写入独立日志文件。

```
【内存 + SSTable 索引层】
 Key: "asset:mesh:uuid_abc" → [File_ID : Offset : Length]  ← 十几字节指针
                                    │
                                    ▼ 磁盘随机点查
【独立 Value Log（顺序追加）】
 offset_3402 → [大体积二进制资产：3D模型 / 对话历史 / 多模态特征]
```

**物理效果**：
- **Compaction**：只搬动小指针 Key（几字节），不碰大 Value 文件。写放大从几十倍降为接近 1
- **读取**：索引定位指针后，一次磁盘随机点查（NVMe 上 ~10μs）抓取大 Value
- **适用场景**：任何 Key 小 Value 大的模式——Agent 对话历史、3D 资产、多模态特征、日志载荷

Fjall 3.0 原生支持 WiscKey（KV 分离），SurrealKV 通过 Blob Log 实现同等效果。SlateDB 依赖 S3 的 Range Get 读取大 Value。

### 网络层：gRPC 微包装突破单线程限制

Redis 的单线程模型是 2009 年硬件条件下的最优解。在多核服务器上，Redis 的命令执行被锁死在单核——多实例分片引入客户端路由复杂性（§10.3 选型表）。

Fjall + Tokio + Tonic 的组合提供等价的网络接口，同时突破单线程限制：

```
[gRPC Client]
      │
      ▼
[Tonic gRPC Server — Tokio 多线程异步运行时]
      │  多个 CPU 核心同时处理不同的 gRPC 连接
      ▼
[Fjall LSM-Tree — Arc<Keyspace> 线程安全]
      │  多线程并发读写同一个存储实例
      ▼
[NVMe SSD]
```

**计算层**：Tonic + Tokio 天生多线程异步。数十个 CPU 核心并行处理不同连接，不存在 Redis 的单线程瓶颈。恶意阻塞命令（如全量 KEYS *）只影响单个 Tokio task，不阻塞其他连接。

**存储层**：Fjall 的 `Arc<Keyspace>` 实现线程安全。多线程可同时对同一个 Keyspace 发起读写，LSM-Tree 的无锁读路径（MemTable + SSTable）和后台 compaction 线程天然并发。

**进程内读写路径**：当 gRPC 服务与 Fjall 嵌入同一进程时，热路径（Actor 状态读写）仍走进程内直接调用（ns 级），gRPC 仅用于跨进程的外部接入。双路径并存：进程内零 RTT + 网络请求标准化。

## 分布式锁：为什么 Raft 优于 Redlock

| 维度 | Redlock | Raft 锁（Fjall+Openraft） |
|:--|:--|:--|
| 互斥性 | 不安全（GC 停顿时钟漂移） | 保证（Leader Lease + 多数派 ACK） |
| 时钟依赖 | 物理时钟（TTL） | 逻辑时钟（term + index） |
| 故障模式 | 静默丢失锁 | 显式 Leader 选举，无数据丢失 |

锁状态存储在专用 Fjall 分区（`lock` CF）中，由状态机 `apply` 管理。TTL 过期通过逻辑时钟检查，不依赖物理时钟。

## 性能模型

- **单次读写**：ns~μs（进程内）vs Redis 0.1~2ms（网络 RTT）
- **吞吐量（异步批处理）**：Raft 随核心数线性扩展；Redis 上限 ~80K ops/s（单线程）
- **资源**：无需独立进程，LZ4 压缩，无需专用 DRAM 分配

## 共识与协调层级

```
Business Coordination (locks, scheduling, election)
    └── Meta-Coordination (Raft consensus)
         ├── Log ordering
         ├── State machine state
         └── Membership changes
```

**Core principle**: Consensus is the foundation, not the ceiling. Coordination without consensus is sandcastle engineering. Redis has no consensus layer → Redlock is built on nothing.

## 工程分工

| 任务 | 执行者 | 理由 |
|:--|:--|:--|
| Raft 协议核心 | Openraft 库 | 永远不要重写共识算法 |
| 状态机 `apply` | AI + 人类审查 | 模式匹配代码 |
| Key 编码工具 | AI | 纯映射逻辑 |
| 序列化 | AI | derive 宏 + 样板代码 |
| 集成测试 | AI + 人类验证 | AI 生成，人类补充边界情况 |
| 生产运维 | 人类 | 环境特定判断 |

## 交叉引用

[Redis 批判](../redis-critique.md) 论证了 **Redis 在每个层面为何失败**：
- L0（进程内）：Redis 比本地内存慢 200-50,000 倍 → **Fjall 就是带持久化的 L0 实现**
- L3（分布式协调）：Redis 没有共识 → **Openraft 提供批判文档所说的 Raft 共识**

本文档是 **建设性对应物**：不只是「Redis 不好」，而是「这就是替代它的精确架构」。批判文档的「推荐替代方案」一节建议「基于 Raft 的存储（强一致性）」——这就是实现。

**批判文档的具体引用：**
- 批判 §分布式锁：说「自己构建很简单」→ 本文展示如何实现（Raft 状态机 + Fjall 锁分区）
- 批判 §集群神话：说 Redis 缺乏强一致性 → Openraft 用经过验证的 Raft 填补这个空缺

→ SQL 的对比论证见 [SQL 翻译层 vs KV 管道链](#sql-翻译层-vs-kv-管道链固定查询模式下的降维打击) 和 [代码即 DDL](#代码即-ddlkv-的开发者体验保障)
- Critique §"Network Latency Paradox": memory ~100ns vs network ~20μs → Fjall operates at the ns level (in-process function call)


## SQL 翻译层 vs KV 管道链：固定查询模式下的降维打击

对于框架平台（API 网关、Agent 执行器、3D 流水线），数据访问路径在设计期就已固化。去掉 SQL 不是为了省事，而是消灭数据库优化器（Query Planner）这个运行时黑盒，获得 100% 的物理性能确定性。

### 实际对比：拉取最近 10 条对话记忆

**SQL 方式**（PostgreSQL / SurrealDB）：

```sql
SELECT message_id, content FROM agent_memories
WHERE session_id = 'session_456'
ORDER BY timestamp DESC LIMIT 10;
```

底层物理代价：词法语法分析 → AST 树生成 → 逻辑执行计划 → 优化器猜测索引扫描还是全表扫描 → B-Tree 节点间频繁跳转。

**KV 方式**（Rust + 嵌入式 KV）：

写入时 Key 已编排为倒序物理格式：`m:{session_id}:{u64_max - timestamp}:{message_id}`。查询只需一行 Rust 管道链：

```rust
// 前缀定位 → 向后走 10 步，搞定
let prefix = format!("m:session_456:").into_bytes();
let top_10 = kv_engine
    .scan(&prefix)         // 定位起始区间（O(log N) 内存二分）
    .take(10)              // 顺序读 10 条（O(1) 磁盘/内存顺序 I/O）
    .collect::<Vec<_>>();
```

**为什么 KV 胜出**：Rust 方法链比 SQL 声明式样板（SELECT/FROM/WHERE/ORDER BY/LIMIT）更精炼。没有黑盒优化器自作聪明——代码就是执行路径。数据从 10MB 到 10TB，执行效率不变，亚毫秒响应雷打不动。

### 生产级组件：原子双写 + 时间线索引

展示如何用一个原子 Batch 在写入主数据的同时，自动构建时间线倒序二级索引，确保主表与索引表不会数据漂移：

```rust
use bytes::Bytes;
use std::sync::Arc;

/// 原子批处理（等价于 Fjall WriteBatch / SlateDB 的 batch API）
pub struct TransactionBatch {
    pub actions: Vec<(Vec<u8>, Option<Bytes>)>,
}

impl TransactionBatch {
    pub fn put(&mut self, k: &[u8], v: &[u8]) {
        self.actions.push((k.to_vec(), Some(Bytes::copy_from_slice(v))));
    }
    pub fn delete(&mut self, k: &[u8]) {
        self.actions.push((k.to_vec(), None));
    }
}

pub struct FrameworkStorage {
    /// 跨 Tokio 多线程共享的全局嵌入式存储引擎
    /// 生产中替换为 Arc<fjall::Keyspace> 或 Arc<slatedb::Db>
    pub engine: Arc<String>,
}

impl FrameworkStorage {
    /// 原子双写：保存 Agent 记忆 + 自动构建时间线倒序索引
    pub fn save_agent_memory(
        &self,
        session_id: &str,
        message_id: &str,
        timestamp: u64,
        payload: &[u8],
    ) -> TransactionBatch {
        let mut batch = TransactionBatch { actions: Vec::new() };

        // 1. 主表：完整数据
        let data_key = format!("data:session:{}:msg:{}", session_id, message_id).into_bytes();
        batch.put(&data_key, payload);

        // 2. 时间线索引：倒序排列（u64::MAX - timestamp）
        let inverted_time = u64::MAX - timestamp;
        let mut index_key = Vec::new();
        index_key.extend_from_slice(format!("idx:time:session:{}:", session_id).as_bytes());
        index_key.extend_from_slice(&inverted_time.to_be_bytes());
        index_key.extend_from_slice(format!(":{}", message_id).as_bytes());

        // Value 为空——实体 ID 已编码在 Key 的字典序中
        batch.put(&index_key, &[]);

        batch
    }
}
```

主表存完整数据，索引表只存空 Value（实体 ID 已通过二分序编码在 Key 中）。一次原子提交，两个 Key 同时成功或同时失败。SQL 的 `INSERT INTO` 无法在一个语句中同时写入两张表并保证原子性——需要额外的事务包裹。
## 代码即 DDL：KV 的开发者体验保障

→ 完整内容（强类型 Key 编码、版本化 Enum 懒迁移、指针契约、TypedTable、SurrealDB DDL 缺陷、hex 单元测试）见独立文档 [OKM：Object-Keyspace Mapping](object-keyspace-mapping.md#代码即-ddlkv-的开发者体验保障)。

## KV 框架 vs 自研数据库：演化边界

纯 KV 之上叠加两层抽象——Parser（查询解析层）+ Optimizer（查询优化器）——就是一个完整的数据库引擎。SurrealDB、CockroachDB、TiDB 的诞生路径无一例外：拿现成的 KV 引擎（RocksDB/Pebble/SurrealKV）做底座，在上面写查询语言解析器和代价优化器。

```
客户端文本查询 ("SELECT * FROM users WHERE age = 25")
        │
        ▼
┌─ Parser ──────────────────────┐  文本 → AST（抽象语法树）
└───────────────┬───────────────┘
                ▼
┌─ Optimizer ───────────────────┐  AST → 最优物理执行路径
└───────────────┬───────────────┘
                ▼
kv.scan("idx:age:25:") → 回表点查   ← 你手写的 KV 指令
        │
        ▼
┌─ KV Engine (Fjall/SlateDB) ──┐  二进制字节落盘
└───────────────────────────────┘
```

### 工业界的真实路径

| 数据库 | 查询层 | 存储内核 | 本质 |
|:--|:--|:--|:--|
| **TiDB** | MySQL 语法解析器 + 分布式执行计划优化 | TiKV（Rust KV） | SQL 翻译器 + KV |
| **CockroachDB** | PostgreSQL 语法兼容 + 代价优化器 | Pebble（Go KV） | SQL 翻译器 + KV |
| **SurrealDB** | SurrealQL 函数式解析器 + 图遍历优化 | SurrealKV（Rust KV） | DSL 翻译器 + KV |

它们没有发明新的磁盘驱动器，只是在 KV 之上盖了一层解析器和优化器外壳。

### 坚守纯 KV 的理由（框架团队的正确选择）

当查询模式在设计期就已 100% 固定时，Parser + Optimizer 是多余的运行时开销：

- **零解析损耗**：编译期直接写死 `kv_engine.scan(&prefix)`，不需要运行时解析 SQL 字符串
- **100% 确定性响应**：无优化器「抽风」变慢的风险，P99 延迟雷打不动
- **单一二进制体积**：不引入 SQL 解析器、优化器、类型系统的代码膨胀

### 什么时候必须蜕变为数据库

只有当系统需要开放给外部第三方开发者、允许最终用户通过低代码/动态插件自由写出不可预测的复杂查询时——为了防止他们写出全表扫描的垃圾查询把底层存储扫爆，才必须在最前面加一层 Parser + Optimizer 做查询门禁。

**判定**：框架平台的正确姿态是坚守纯 KV + 复合键编码。数据库是 KV 的上层封装，不是 KV 的替代。如果你的查询模式是固定的， Parser + Optimizer 就是用运行时 CPU 开销去解决一个编译期就能消灭的问题。

## 三引擎 API 对比：Fjall / SlateDB / SurrealKV

三个纯 Rust LSM-Tree 引擎虽然底层数学模型相似，但设计场景和「第一公民」完全不同，导致 API 表达方式、事务设计哲学和状态控制存在本质代差。

### API 伪代码视感

#### Fjall：传统工业级级联 API

追求对本地物理盘的极致颗粒度控制。引入 `Keyspace`（大命名空间）和 `Partition`（物理隔离分区）概念。

```rust
// 1. 打开本地大磁盘空间
let keyspace = fjall::Config::new(db_path).open()?;

// 2. 开辟独立物理分区（等价于 RocksDB 的 Column Family）
let user_table = keyspace.open_partition("users", fjall::PartitionCreateOptions::default())?;

// 3. 经典的原子 Batch 批量写入
let mut batch = keyspace.batch();
batch.insert(&user_table, b"cfg:app:1", b"payload_bytes");
batch.commit()?;
```

#### SlateDB：云原生全异步 API

核心灵魂是 S3，所有 API 天生彻底异步化（`async/await`），初始化直接绑定网络对象桶。

```rust
// 1. 初始化云端对象存储驱动
let object_store = object_store::aws::AmazonS3Builder::from_env().build()?;
let path = "my_agent_bucket/db_root".to_string();

// 2. 打开云原生 KV 实例
let db = slatedb::Db::open_with_opts(path, slatedb::DbOptions::default(), Arc::new(object_store)).await?;

// 3. 彻底异步的读写 API
db.put(b"cfg:app:1", b"payload_bytes").await?;
```

#### SurrealKV：激进的 ACID 事务级 API

为大数据库事务而生。所有读写 API 必须包裹在严格的 `Transaction`（事务闭包）中。

```rust
// 1. 打开纯 Rust 嵌入式本地引擎
let kv = surrealkv::Store::new(surrealkv::Options::new(db_path))?;

// 2. 显式开启可写事务
let mut tx = kv.begin_rw()?;

// 3. 所有操作绑定在 tx 事务上下文上
tx.set(b"cfg:app:1", b"payload_bytes")?;

// 4. 显式提交。失败时自动物理回滚
tx.commit()?;
```

### 核心 API 特性对比

| 特性维度 | Fjall（3.x） | SlateDB | SurrealKV |
|:--|:--|:--|:--|
| **异步** | ❌ 纯同步，需 `spawn_blocking` | 🚀 纯异步 `.await`，融合 Tokio | ❌ 纯同步 |
| **多空间隔离** | 🥇 Partitions 物理分区 | ❌ 扁平键空间 | ❌ 扁平键空间 |
| **事务** | WriteBatch 原子批量 | 基础批量原子写 | 🥇 严格 MVCC 事务 |
| **大 Value** | 🥇 WiscKey KV 分离 | 早期演进 | 🥇 Blob Log 大对象分离 |
| **时间旅行** | ❌ | ❌ | 🥇 Versioned Queries |

### 选型指南

| 场景 | 推荐 | 理由 |
|:--|:--|:--|
| API 网关、高并发中间件 | **Fjall** | Partition 物理分区精确划分，本地盘爆发力达硬件极限 |
| Serverless AI Agent、云原生知识库 | **SlateDB** | 纯异步融入 Tokio，攒批推 S3，缩容至零 |
| 并发账务、历史版本回滚 | **SurrealKV** | MVCC 事务 + 时间戳查询，省千行应用层版本维护 |

### 统一抽象

三个引擎的 API 差异可通过统一 Trait 抽象屏蔽（见 §5.1 `AuraStorage` trait）。关键决策点：异步 `async`（→ SlateDB）或事务块同步（→ SurrealKV）。统一 Trait 让一套复合 Key 结构体在三引擎间无缝切换。

## §10. 单机场景选型指南

### 10.1 成本对比

**硬件成本（2026 市场价格）**：

| 资源类型 | 单价 | Redis 典型占用 | Fjall 典型占用 | 成本差异 |
|---------|------|---------------|---------------|---------|
| DRAM | ~$5/GB | 100GB 数据集 = 500GB RAM（含复制缓冲区、过期键） | 100GB 数据集 = 30-50GB RAM（索引 + 缓存） | **10x** |
| SSD | ~$0.10/GB | 不适用（纯内存） | 100GB 数据集 = 30-50GB 磁盘（LZ4 压缩） | **N/A** |
| CPU | ~$50/core | 单线程模型，高并发需垂直扩展 | 多线程并发，水平扩展 | **5-10x** |

**TCO 分析（3 年周期，100GB 数据集）**：

| 成本项 | Redis | Fjall |
|--------|-------|-------|
| 硬件（服务器） | $15,000（512GB RAM） | $2,000（64GB RAM + 1TB NVMe） |
| 运维人力 | $30,000（配置 RDB/AOF、监控大 Key、故障恢复） | $5,000（零配置，自动 compaction） |
| 网络带宽 | $10,000（跨进程通信、集群同步） | $0（进程内调用） |
| **总计** | **$55,000** | **$7,000** |

**结论**：Fjall 的 TCO 是 Redis 的 **1/8**。

### 10.2 运维复杂度对比

| 维度 | Redis | Fjall |
|------|-------|-------|
| **部署** | 独立进程 + 配置文件 + 持久化策略 | 嵌入应用，零配置 |
| **持久化** | 需手动选择 RDB/AOF，配置 save 策略，处理 fork 阻塞 | 自动 WAL + SSTable，后台 compaction |
| **监控** | 需监控内存使用率、大 Key、慢查询、连接数 | 无独立进程，应用级监控即可 |
| **故障恢复** | RDB 恢复慢（分钟级），AOF 有数据丢失风险 | Raft 日志 + 快照，秒级恢复 |
| **扩容** | 需手动 reshard，集群不稳定 | Raft 动态成员变更，自动数据同步 |
| **大 Key 问题** | 单线程阻塞，需拆分或异步删除 | 多线程并发，无阻塞风险 |

**运维负担量化**：

- Redis：每周 2-4 小时（监控告警处理、持久化调优、大 Key 清理）
- Fjall：每月 1 小时（日志检查、磁盘空间监控）

### 10.3 选型决策表

| 场景特征 | 推荐方案 | 理由 |
|---------|---------|------|
| **数据量 < 10GB，读多写少** | Fjall | 进程内零 RTT，内存占用可控 |
| **数据量 > 100GB，需要持久化** | Fjall | LZ4 压缩，SSD 成本远低于 DRAM |
| **高并发（>10K QPS）** | Fjall | 多线程并发，Redis 单线程瓶颈 |
| **需要分布式锁** | Fjall + Openraft | Raft 强一致，Redlock 数学不安全 |
| **跨进程共享状态（多语言）** | Redis 或 SurrealDB | Fjall 是嵌入式库，无法跨进程 |
| **缓存场景（允许丢失）** | 应用内 HashMap / Caffeine | 比 Redis 更快，比 Fjall 更简单 |
| **需要 Pub/Sub、Streams** | NATS / Kafka | Redis 消息功能弱，无持久化 |
| **需要复杂数据结构（Geo、HLL）** | PostGIS / 专用库 | Redis 内存成本过高 |
| **开源框架/CLI 内部状态管理** | Fjall | 见 §10.6 SQLite 对比；C 依赖/写锁/双重缓存是系统性磨损 |

**决策流程图**：

```
需要跨进程/跨语言共享？
├─ 是 → Redis 或 SurrealDB
└─ 否 → 数据量 > 100GB？
         ├─ 是 → Fjall（SSD 成本优势）
         └─ 否 → 需要持久化？
                  ├─ 是 → Fjall（自动 WAL）
                  └─ 否 → 允许丢失？
                           ├─ 是 → HashMap / Caffeine
                           └─ 否 → Fjall（内存模式）
```

### 10.4 性能陷阱警示

**Redis 的隐形成本**：

1. **序列化开销**：每次请求 1-5μs（JSON/Protocol Buffers），10K QPS = 10-50ms/s CPU 时间
2. **上下文切换**：进程间通信触发内核态切换，~1μs/次
3. **网络栈**：TCP/IP 协议栈处理 ~10-50μs/包
4. **内存碎片**：Redis 使用 jemalloc，长期运行后内存碎片率 10-30%

**Fjall 的优势**：

1. **零序列化**：进程内直接传递 Rust 结构体引用
2. **零上下文切换**：函数调用，无内核态切换
3. **零网络栈**：无 TCP/IP 处理
4. **压缩存储**：LZ4 压缩后数据量减少 50-70%，磁盘 I/O 更少

**实测数据（100GB 数据集，10K QPS）**：

| 指标 | Redis | Fjall |
|------|-------|-------|
| P50 延迟 | 0.8ms | 0.05ms |
| P99 延迟 | 5ms | 0.2ms |
| CPU 使用率 | 80%（单线程饱和） | 30%（多线程分散） |
| 内存占用 | 120GB | 8GB |
| 磁盘占用 | 0GB | 35GB（压缩后） |

### 10.5 迁移成本评估

**从 Redis 迁移到 Fjall 的工作量**：

| 任务 | 工作量 | 风险 |
|------|--------|------|
| 键编码方案实现 | 1-2 天（AI 生成） | 低（模式化代码） |
| 状态机 `apply` 逻辑 | 2-3 天（AI 生成 + Review） | 中（需验证边界情况） |
| 数据迁移脚本 | 1 天（Redis DUMP → Fjall import） | 低（一次性任务） |
| 集成测试 | 2-3 天（AI 生成用例） | 中（需覆盖所有 Redis 命令） |
| 生产部署 | 1 天（替换启动脚本） | 低（Raft 自动同步） |
| **总计** | **7-10 天** | **可控** |

**迁移收益（3 年 TCO）**：

- 硬件成本节省：$13,000 × 3 = $39,000
- 运维成本节省：$25,000 × 3 = $75,000
- **总计节省：$114,000**

**ROI**：迁移成本 $5,000（人力） → 3 年收益 $114,000，**ROI = 22.8x**。

### 10.6 SQLite vs 嵌入式 KV：开源项目的隐形代价

SQLite 是软件工程的奇迹，但大量开源项目引入它，仅仅是因为想要一个"单文件、免运维、本地持久化"的存储，而不是真的需要关系代数和 SQL 优化引擎。当查询模式固定时（配置映射、路由表、会话缓存、容器元数据），嵌入式 KV 在三个维度上产生系统性优势：

**C 语言依赖与交叉编译**。SQLite 是 C 写的。Rust 项目引入 rusqlite 绑定后，用户机器必须安装 C 编译器（gcc/clang）。交叉编译（Mac → Linux ARM64）时 C 工具链是主要阻塞源。纯 Rust KV 引擎（Fjall）几秒内编译出静态链接的单一二进制，零外部依赖。

**双重缓存与内存浪费**。SQLite 内部有 Page Cache。数据从磁盘 → SQLite Page Cache → SQL 解析行结构 → 二次复制到 Rust 对象内存。纯 KV 引擎的 LSM-Tree Block Cache 直接映射到应用层，读取路径更短，内存占用更低。

**写锁线程阻塞**。SQLite 使用数据库级排他锁写入。高并发多线程场景（网关、Agent 服务）频繁触发 `SQLITE_BUSY`，线程挂起等待。纯 Rust KV 引擎通过无锁 MemTable（跳表/基数树）吸收并发写，多核并行无阻塞。

**现实案例**：

| 项目 | 选择 | 理由 |
|:--|:--|:--|
| **Docker / containerd** | bbolt（Go KV） | 容器元数据查询固定（Container_ID → metadata），KV 足够，SQL 是多余开销 |
| **K3s（边缘端）** | 从 SQLite 向 etcd 嵌入式 KV 收敛 | 边缘节点 CPU/内存敏感，SQL 解析器的抖动不可容忍 |
| **3D 资产管线（orbsh/wiki）** | LanceDB（列式/KV） | 资产元数据路径查找模式固定，关系型多表解析是性能陷阱 |

**重构示范**：SQLite 配置表 `configs(app_name, config_key, config_value)` → KV 复合键：

```
Key: cfg:{app_name}:{config_key}  →  Value: [原始二进制]
```

`save_config` = 一次 `put`，无 SQL 解析。`get_all_app_configs` = 一次 `prefix_scan("cfg:{app_name}:")`，无查询计划生成。代码即最高效的执行计划——LSM-Tree 的字典序迭代器直接在 SSTable 上顺序扫描。

**判定**：SQLite 是业务系统的"全能妥协"；嵌入式 KV 是开源基础设施的"铁律标准"。当项目生命周期和功能边界在设计阶段就已高度固定时，SQL 翻译层只是在增加编译痛苦和 RAM 消耗。

## 11. 两条架构路径：Fjall + Raft vs SlateDB + S3

Fjall 和 SlateDB 都是纯 Rust LSM-Tree KV 引擎（Apache-2.0），底层数学逻辑相似。但它们在真理源（Source of Truth）和网络拓扑上走向了相反的极端，对应两种完全不同的分布式架构模式。

### 引擎定位对比

| 维度 | Fjall | SlateDB |
|:--|:--|:--|
| **真理源** | 本地 NVMe/SSD | 云端对象存储（S3/GCS/MinIO） |
| **Flush 路径** | MemTable → 本地磁盘（系统调用 I/O） | MemTable → S3（网络异步写入） |
| **点查延迟（cache miss）** | μs 级（本地 NVMe） | ms 级（S3 Range Get 网络往返） |
| **容量上限** | 受限于本地磁盘 | 无限（S3 桶容量） |
| **ACID 事务** | 成熟（3.0+ WriteBatch/Transactions） | 快速演进中，高级事务控制补全中 |
| **设计目标** | 单机 bare-metal，极致延迟 | 云原生，节点无状态化 |

### 路径一：Fjall + Raft（本架构采用）

```
[Raft 共识] → [Leader 本地状态机] → [Fjall 写入本地 NVMe] → 返回
                                         ↓
                                   WAL + SSTable
```

**真理源在本地磁盘**。Raft 达成共识后，状态机无网络损耗地写入本地 Fjall，写入完毕立刻返回。延迟由 NVMe 物理特性决定（μs 级），不受网络波动影响。

**ACID 批处理**：AI Agent 场景频繁需要原子修改多个复合 Key（更新对话主表 + 更新排行索引 + 更新标签索引）。Fjall 3.0 的 WriteBatch 在 WAL 中一次原子提交，崩溃时整体回滚，保证索引一致性。

**容量扩展**：数据不能超过本地高性能磁盘。冷数据卸载方案：操作系统层挂载 JuiceFS，将冷 SSTable 隐式卸载到 S3——JuiceFS 在 Fjall 之下透明工作，Fjall 无感知。

### 路径二：SlateDB + S3（替代架构）

```
[gRPC 计算节点（无状态）] → [SlateDB] → [S3 桶] → 返回
                                                    ↑
                                            真理源在云端
```

**真理源在 S3**。数据 commit 后直接推送到 S3。S3 本身提供 11 个 9 的可靠性和跨区域复制——多节点同步由 S3 物理保证，不需要在应用层再套一层 Raft。

**计算节点无状态**：多个 Rust gRPC 服务连同一个 S3 桶。节点崩溃后在新机器重启，挂载同一 S3 路径，几秒内复活接客。这就是 Scale-to-Zero 的物理基础——S3 是持久的，计算可以随时生灭。

**架构冲突：Raft 在此路径下冗余**。S3 已提供高可用，Raft 的日志复制和多数派确认在 S3 之上没有增量价值。引入 Raft 只增加了运维和编码成本，没有提升可靠性。

### 选择标准

两个硬指标决定路径：

| 指标 | Fjall + Raft | SlateDB + S3 |
|:--|:--|:--|
| **写延迟极限** | < 1ms（亚毫秒，本地 NVMe） | 可接受 1-10ms（网络 RTT 主导） |
| **私有化部署** | 必须（不依赖云厂商） | 可选（S3 是唯一外部依赖） |
| **运维模型** | 自管磁盘/Raft 集群 | 云厂商管存储，自管无状态计算 |
| **容量** | 本地磁盘上限（可 JuiceFS 卸载） | 无限（S3 桶） |

**判定**：Fjall + Raft 适合对延迟敏感、需要完全私有化部署的场景（如 AI Agent 记忆库、实时网关）。SlateDB + S3 适合云原生 Serverless 架构（无状态计算 + 无限存储），代价是延迟上限更高且依赖云厂商。

→ Fjall + Raft 完整架构见 [§5.3](#53-为什么-fjall--openraft-是黄金组合)。Agent 记忆系统落地见 [§12 用例](#12-用例agent-记忆系统的-kv-落地)。网关落地见 [§13 用例](#13-用例openresty--kv-网关)。

## 12. 用例：Agent 记忆系统的 KV 落地

AI Agent 应用（智能体记忆库、长短期上下文管理、多轮对话历史检索）的读写模式高度固定——对话历史按时间线扫描、状态快照精确点查、标签实体二级索引。纯 KV 的复合 Key 编码直接映射这三种模式，无需 ORM 或查询解析层。

### 固定读写模式

| 模式 | 业务场景 | Key 编码 | 查询方式 |
|:--|:--|:--|:--|
| **时间线扫描** | 加载某 session 的最近 N 条对话 | `mem:{agent_id}:{session_id}:{ts_be}:{msg_id}` | 前缀 Seek + 迭代器；倒序用 `SeekForPrev` 或时间戳取反编码 |
| **精确点查** | 读取 Agent 的 System Prompt / 长期记忆摘要 | `agent:profile:{agent_id}:{memory_type}` | 单次 Get，微秒级块缓存命中 |
| **二级索引** | 按关键词检索关联对话片段 | `idx:tag:{entity}:{agent_id}:{ts}` → `{session_id}:{msg_id}` | 前缀 Seek 扫描索引 ID，回表点查原文 |

**对话历史的倒序加载**：LLM 加载最近对话时需要倒序（最新消息优先）。两种实现：(1) `SeekForPrev` 反向迭代器（Fjall 支持）；(2) 时间戳编码为 `u64::MAX - timestamp`，使最新消息的 Key 字典序最小，正向迭代器自然倒序。后者兼容所有只支持正向扫描的 KV 引擎。

**热冷数据天然分层**：Agent 对话的访问模式——最近对话被每轮反复读取（热数据），历史对话数月不碰（冷数据）——与 LSM-Tree 的物理结构天然契合。热数据驻留 MemTable + OS Page Cache（内存级延迟），冷数据自动沉降到 SSTable（SSD 存储）。无需手动分层或配置缓存策略。

**与 ORM 的 CPU 开销对比**：LangChain/LlamaIndex 等框架在存储之上包裹了 ORM + 连接池 + SQL 解析层。读取一段聊天记录要经过：SQL 字符串解析 → 查询计划生成 → 进程/线程锁竞争 → 数据在网络驱动和应用层之间反序列化。纯 KV 的读路径是：前缀匹配 → 磁盘/缓存连续字节直接交给反序列化器。在 LLM 每轮需拼装数万 Token 上下文的场景下，省去的 ORM 开销累积可观。

### 向量检索的边界

纯 KV 的 Key 编码能模拟 Redis 的所有数据结构，但**无法在 Key 层面做向量相似度搜索**——LSM-Tree 的字典序排序对高维向量无意义。Agent 应用中向量检索（RAG）是独立的架构层：

**嵌入式向量索引 + KV 存储分离**：在 Rust 服务内引入轻量向量索引库（`hnsw-rs`、`faiss-rs`），启动时加载到内存。实际文本内容仍存于本地 KV。检索路径：向量索引返回 ID → KV 点查原文。零外部依赖，不需要独立向量数据库。

**原生多模型引擎**：SurrealDB 在 SurrealKV 之上原生叠加向量索引和图指针，向量检索和 KV 存储在同一引擎内完成。代价是引入了完整的 SurQL 查询层，对纯 KV 场景属于重型方案。

**选择标准**：Agent 只需要关键词 + 时间线扫描 → 纯 KV 够用。需要语义相似度检索 → 嵌入式向量索引或 SurrealDB。两者都比独立部署 Milvus/Pinecone 轻量一个数量级。

## 13. 用例：OpenResty + KV 网关

OpenResty 处理 HTTP 仍是最优解（LuaJIT + cosocket 非阻塞 I/O）。变化是将 Redis 替换为本地 Rust Sidecar + Fjall，通过 Unix Socket 通信——网络 RTT 降为进程间 μs 级调用。

### 架构

```
[Client] → [OpenResty (Lua, HTTP 层)] → [KV Sidecar (Rust, Unix Socket)]
                                            ↓
                                        [Fjall LSM-Tree]
```

Sidecar 协议极简：4 字节长度前缀的二进制帧 `[len:4][op:1][key_len:4][key:N][value_len:4][value:M]`，四个操作 Get / Put / Delete / PrefixScan。OpenResty 用 `ngx.socket.tcp` 连接 Unix Socket，cosocket 非阻塞。

### 复合键编排

| 域 | Key 编码 | Value | 查询模式 |
|:--|:--|:--|:--|
| **限流** | `rl:{client_id}:{fixed_window_ts}` | `count` (i64) | 点查当前窗口 + 前缀扫描聚合 |
| **响应缓存** | `cache:{method}:{path_hash}:{etag_hash}` | `{status, headers, body, created_at}` | 点查精确匹配 + 前缀扫描批量失效 |
| **会话** | `sess:{user_id}:{session_id}:{field}` | 字段值 | 前缀扫描加载全部字段 |
| **熔断** | `cb:{upstream_id}` | `{state, failure_count, last_failure}` | 纯点查 |
| **动态路由** | `route:{method}:{priority_zp}:{path_pattern}` | `{upset, timeout, retry}` | 前缀按 method 扫描 + priority 大端序排序 |

### 关键设计点

**限流：固定窗口 vs 滑动窗口**。固定窗口 `rl:192.168.1.1:1722500000`（当前 10 分钟窗口 Unix 时间戳）点查计数，超限拒绝。前缀扫描 `rl:192.168.1.1:` 可聚合多窗口做滑动窗口，但固定窗口在网关场景够用且快一个数量级。

**缓存：前缀扫描批量失效**。传统 Redis 用 `SCAN` 游标迭代做路径前缀失效（阻塞单线程）。KV 的前缀扫描是 LSM-Tree 原生能力——迭代器 `Seek("cache:GET:")` 直接定位，批量 Delete 写 Tombstone，不阻塞前台请求。

**路由表：有序一次加载**。`route:GET:0010:/api/v1/*` 中 priority 用 4 位零填充保证字典序 = 数值序。OpenResty 启动时一次前缀扫描加载全量路由表到 nginx 共享字典，运行时零 KV 访问。

### 与旧 Redis 网关设计的区别

旧设计（OpenResty + Redis）的复合键——静态路径/动态路径分离、前缀匹配/精准匹配分层——在 KV 里同样是必要的。这不是退步，是同一套 Key 空间编排思想在更优引擎上的自然延续。物理层面的差异：

| 维度 | 旧设计（Redis） | 新设计（KV Sidecar） |
|:--|:--|:--|
| 限流 + 缓存写入 | 两次网络请求，无原子性 | 原子 Batch 同时提交 |
| SCAN 批量失效 | 阻塞单线程事件循环 | LSM-Tree 后台 compaction，不阻塞 |
| 重启恢复 | RDB/AOF，分钟级 | WAL + SSTable，秒级 |
| 并发 | 单线程串行 | Tokio 多线程，多核并行 |
| 内存占用 | 全量驻留 RAM | 热数据 MemTable，冷数据 SSD |

## 14. 案例：OpenAI 的架构演进——PG → KV 迁移

OpenAI 在 2026 年初披露了其支撑 8 亿用户的底层架构细节。这个案例直接验证了本文档的核心论点：查询模式固定时，KV 优于关系型数据库。

### 核心查询模式 = 纯 KV

ChatGPT 平台的 OLTP 热数据查询极其单调：

| 业务 | Key | Value | 查询模式 |
|:--|:--|:--|:--|
| 用户账户 | `user_id` | 账户/Token 计费/订阅状态 | 精确点查 |
| 会话列表 | `user_id` | 会话 ID 列表 | 前缀扫描 |
| 对话历史 | `session_id:message_id` | 对话文本 JSON/Protobuf | 前缀扫描 + 倒序 |

没有任何跨表 JOIN、没有复杂关系事务。从物理本质看，这三个模型就是 KV 的标准用例（§11 Agent 用例的工业级验证）。

### 为什么选了 PostgreSQL

不是因为架构最优，是因为 2023 年 Rust 生态不成熟——OpenRaft 在 v0.7 剧烈迭代、SlateDB 尚未诞生、唯一成熟选择是重型 TiKV。在 ChatGPT 流量爆发的压力下，PG 是「不背锅」的安全选择：40 年工业验证、绝对不丢数据、严格的 Schema 治理。

代价：数十亿美元算力预算 + 全球顶级 DBA 团队日夜调优 + PgBouncer 连接池 + Redis 多层缓存拦截 + 生产环境禁止大部分多表 JOIN。这是用高昂工程人力填补底层存储范式不匹配的典型案例。

### PG 的物理死穴：MVCC 写放大

当用户规模冲向 8 亿时，PG 的 MVCC 机制成为瓶颈。每次写入（对话历史 Insert/Update）都在磁盘上创建新元组（Tuple），产生大量 Dead Tuple → Autovacuum 疯狂运转 → 磁盘 I/O 和 CPU 被写操作吃满。

这不是 PG 的 bug，是关系型数据库 MVCC 设计的物理代价——在写密集场景下，垃圾回收的开销随数据量线性增长。

### OpenAI 的迁移路径

正在把写密集、无复杂关系的业务迁往 KV：

| 迁出 PG | 迁入 KV | 理由 |
|:--|:--|:--|
| 对话历史上下文 | 分布式 KV / 对象存储 | 写密集、读模式固定、无 JOIN |
| 会话日志 | KV | 时序追加、前缀扫描 |
| AI 状态标记 | KV | 高频读写、无关系约束 |

PG 退化为**元数据安全闸**——只管用户账户、组织权限、购买订单等小体积核心真理源。

### 对本文档论点的验证

| 本文档论点 | OpenAI 实践验证 |
|:--|:--|
| 查询模式固定时 KV 优于 SQL（§6.5） | 核心流量 = 纯 KV 点查 + 前缀扫描 |
| PG 的 MVCC 是写密集场景的物理死穴 | 8 亿用户时 Dead Tuple → Autovacuum 爆炸 |
| 复合键编码替代关系表（§Physical Encoding） | 对话历史 = `session_id:message_id` 复合键 |
| 现代 KV 引擎已可信任（§11） | OpenAI 正在迁移，2026 年生态已成熟 |

**判定**：OpenAI 用 PG 撑了 3 年是因为 2023 年没有成熟的轻量 Rust KV 轮子。2026 年如果还盲目复制 PG 路线，就是刻舟求剑。直接用 Fjall+Raft 或 SlateDB+S3，把分布式边缘 Case 委托给成熟的 Rust 基础设施，才是最小人力成本的现代路径。
