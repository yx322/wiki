---
title: OKM：Object-Keyspace Mapping
created: 2026-07-21
updated: 2026-07-21
type: concept
tags: [composite-key, embedded-kv, rust, architecture, performance]
sources: [raw/articles/object-keyspace-mapping.md]
confidence: high
---

# OKM：Object-Keyspace Mapping

OKM 是对标 ORM 的范式——ORM 将对象映射到关系表，OKM 将对象映射到 KV 键空间。通过自定义过程宏 `#[derive(KvEncode)]` + 数字命名空间 ID，构建零成本抽象语义数据层。

## 核心理念：代码即 DDL

SQL 定义表结构，Rust Struct 定义 Key 编码。编译器保证格式一致，脏数据在 `cargo check` 阶段就被熔断。

## 五大组件

### 1. 强类型 Key 编码（Struct 即 DDL）

```rust
pub struct UserSessionKey {
    pub org_id: [u8; 16],     // 固定 16 字节 UUID
    pub user_id: [u8; 16],    // 固定 16 字节 UUID
    pub session_id: [u8; 16], // 固定 16 字节 UUID
}
// Key 布局：sess: (5) + org (16) + user (16) + session (16) = 53 字节
```

所有字段固定宽度，反序列化纯指针切片，零解析。

### 2. 版本化 Enum 懒迁移（替代 ALTER TABLE）

```rust
pub enum MemoryValuePayload {
    V1(AgentMemoryV1),
    V2(AgentMemoryV2),  // 新增字段
}
```

只有被读到的老记录才升级，未读到的继续旧格式存储。SQL `ALTER TABLE` 必须遍历全表重写。

### 3. Edge Struct（模拟外键）

正向 + 反向双写 Key 维护 1:N 关系：
- 正向：`edge:s2m:{session_id}:{message_id}`
- 反向：`edge:m2s:{message_id}:{session_id}`

### 4. TypedTable（泛型类型安全抽象）

`TypedTable<UserSessionKey, SessionData>` 在编译期锁死 Key 和 Value 类型，传错类型直接编译失败。

### 5. DDL 稳定性单元测试

硬编码 hex 字节锁定 Key 编码，任何改动 CI/CD 立刻阻断：

```rust
let expected = hex::decode("736573733a...").unwrap();
assert_eq!(test_key.encode(), expected,
    "DDL 物理 Key 编码发生非预期漂移！");
```

## 数字命名空间字典（85% 前缀压缩）

字符串前缀 `"user_sessions:"` (14 字节) → 压缩为 `u16` 数字 ID (2 字节)。Map 极小，100% 常驻 CPU L1 Cache。

```rust
#[derive(KvEncode)]
#[kv_ns(1)]  // 编译期翻译为 [0x00, 0x01] 2 字节前缀
pub struct UserSessionKey {
    pub org_id: [u8; 16],
    pub user_id: u64,
    pub session_id: [u8; 16],
}
// 物理总长度 = 2 + 16 + 8 + 16 = 42 字节
```

## 过程宏 `#[derive(KvEncode)]`

编译期自动：
1. 提取 `#[kv_ns(N)]` 数字 → 2 字节大端序前缀
2. 遍历字段，根据类型生成 `extend_from_slice` 代码
3. 计算总固定宽度，生成 `decode()` 指针切片偏移
4. **零运行时开销**——编译后固化为机器码

### syn→quote 类型映射

| 字段类型 | syn 分析 | quote 生成 | 字节宽度 |
|---------|---------|-----------|---------|
| `[u8; N]` | `Type::Array` | `extend_from_slice` | N（固定） |
| `u64` / `i64` | `Type::Path` | `to_be_bytes()` | 8 |
| `u32` / `i32` | `Type::Path` | `to_be_bytes()` | 4 |

## TypedCollection（ORM 级开发体验）

```rust
pub struct TypedCollection<T> {
    _marker: PhantomData<T>,           // 编译期类型锁死，运行时 0 字节
    pub raw_backend: Arc<dyn AuraStorage>,
}

// 业务代码
collection.save(&mut batch, &data);      // 保存
collection.find_by_key(&key_spec);       // 查询
```

宏自动分离 Key 字段和 Value 字段：
- Key 字段 → `generate_compised_key()` 编排为定长二进制
- Value 字段 → `bincode::serialize()` 序列化

## 物理收益

- **85% 前缀压缩**：14 字符串 → 2 数字字节
- **100% 定长 Key**：反序列化纯指针切片，零解析
- **65535 命名空间**：u16 覆盖全场景
- **Cache Locality 极致**：CPU 缓存局部性极好
- **零运行时开销**：无正则/split/AST

## 与 SurrealDB/Mongo 的 DDL 缺陷对比

无模式（Schema-less）是"致幻剂"：
- 弱约束导致应用层防御性代码爆炸
- 查询优化器无法预判数据结构
- OKM 在编译期熔断脏数据，既消灭无模式风险，又白嫖 KV 硬件速度

## Openraft 状态机集成

OKM 编码的实体通过 Raft 广播后，状态机将其 Apply 到 [[fjall]]：

```rust
RaftCommand::UpdateActorState { agent_id, serialized_context }
// serialized_context 可以是 OKM 编码的实体
```

## 关联页面

- [[fjall]] — 底层存储引擎
- [[composite-key-encoding]] — 复合键编码基础
- [[application-mvcc]] — 版本化 Enum 实现 MVCC
- [[raft-consensus]] — Openraft 集成
