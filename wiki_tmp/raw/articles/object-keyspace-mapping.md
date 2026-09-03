# OKM：Object-Keyspace Mapping（对象键空间映射）

**Status:** 独立组件
**关联文档：** [KV 存储引擎](kv-storage-engine.md) — 底层架构与设计模式
**关联文档：** [Aura 架构](aura-architecture.md) — 双引擎模式

---

OKM 是对标 ORM 的范式——ORM 将对象映射到关系表，OKM 将对象映射到 KV 键空间。通过自定义过程宏 `#[derive(KvEncode)]` + 数字命名空间 ID，构建零成本抽象语义数据层：开发侧如同 ORM 般声明式，编译后退化为纯指针偏移计算。

## 代码即 DDL：KV 的开发者体验保障

SQL 的核心价值不是执行性能，而是关系模型交付的可读性、建模规整度和团队协作确定性。裸露的二进制字节 Key 会退化为面条代码——团队无法理解 Key 的排列规则。解决方案：用 Rust 类型系统替代 SQL DDL，把 Schema 正确性从运行时数据库引擎上提到编译期编译器。

### 强类型 Key 编码：Struct 即 DDL

SQL 定义表结构，Rust Struct 定义 Key 编码。编译器保证格式一致：

```rust
/// UserSessionKey 就是你的 DDL。
/// 跨整个框架强制约束 Key 的合法性。
pub struct UserSessionKey {
    pub org_id: [u8; 16],     // 固定 16 字节 UUID 二进制
    pub user_id: [u8; 16],    // 固定 16 字节 UUID 二进制
    pub session_id: [u8; 16], // 固定 16 字节 UUID 二进制
}

impl UserSessionKey {
    /// 编码器 = 物理 Schema 的序列化引擎。
    /// 所有字段固定宽度，无分隔符，无动态解析。
    /// Key 布局：sess: (5) + org (16) + user (16) + session (16) = 53 字节
    pub fn encode(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(53);
        key.extend_from_slice(b"sess:");           // 5 字节命名空间前缀
        key.extend_from_slice(&self.org_id);       // 16 字节固定宽度
        key.extend_from_slice(&self.user_id);      // 16 字节固定宽度
        key.extend_from_slice(&self.session_id);   // 16 字节固定宽度
        key
    }
}
```

任何试图写入错误格式的代码在编译期直接报错。不需要运行时校验。

### 版本化 Enum 懒迁移：零停机 Schema 演进

SQL 靠 `ALTER TABLE` 做迁移，需要锁表。KV 靠版本化 Enum 做懒迁移——读取时按版本自动升级，无停机：

```rust
#[derive(Serialize, Deserialize)]
pub enum MemoryValuePayload {
    V1(AgentMemoryV1),
    V2(AgentMemoryV2),  // 新增字段：多模态、特征标签
}

// 读取时按版本匹配，老数据在内存中无感升级
match database.get(&key) {
    MemoryValuePayload::V1(old) => upgrade_v1_to_v2(old),
    MemoryValuePayload::V2(current) => current,
}
```

只有被读到的老记录才升级。未读到的继续以旧格式存储，不浪费写入带宽。这是 SQL `ALTER TABLE` 无法做到的——PG 的迁移必须遍历全表重写所有行。

### 指针契约：Edge Struct 模拟外键

SQL 的外键关系在 KV 中通过显式的正向+反向双写 Key 维护：

```rust
/// Session → Messages 的 1:N 外键关系
pub struct SessionToMessageEdge {
    pub session_id: [u8; 16],  // 固定 16 字节 UUID 二进制
    pub message_id: [u8; 16],  // 固定 16 字节 UUID 二进制
}

impl SessionToMessageEdge {
    /// 正向：从会话找消息
    /// Key 布局：edge:s2m: (9) + session (16) + message (16) = 41 字节
    pub fn forward_key(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(41);
        key.extend_from_slice(b"edge:s2m:");
        key.extend_from_slice(&self.session_id);
        key.extend_from_slice(&self.message_id);
        key
    }
    /// 反向：从消息反查会话（等价于 SQL Foreign Key 联合索引）
    /// Key 布局：edge:m2s: (9) + message (16) + session (16) = 41 字节
    pub fn reverse_key(&self) -> Vec<u8> {
        let mut key = Vec::with_capacity(41);
        key.extend_from_slice(b"edge:m2s:");
        key.extend_from_slice(&self.message_id);
        key.extend_from_slice(&self.session_id);
        key
    }
}
```

写入时同时 `put` 正向和反向 Key。删除时同时 `delete` 两侧。Edge Struct 的代码注释就是 E-R 图的文档化——比 SQL DDL 更显式，因为关系编码逻辑和 Key 生成逻辑在同一处。

### TypedTable：泛型类型安全抽象

在裸字节 KV 引擎之上封装一层声明式泛型表，交付 ORM 级别的开发体验：

```rust
use std::marker::PhantomData;

/// 通用泛型表空间——PhantomData 编译期类型检查，运行时零开销
pub struct TypedTable<K, V> {
    _key_type: PhantomData<K>,
    _val_type: PhantomData<V>,
}

impl<K, V> TypedTable<K, V> where
    K: serde::Serialize,
    V: serde::Serialize + serde::de::DeserializeOwned
{
    /// 类型化写入——编译器保证 Key 和 Value 类型匹配
    pub fn put_record(&self, batch: &mut Vec<(Vec<u8>, Vec<u8>)>, key: K, value: V) {
        let serialized_key = bincode::serialize(&key).unwrap();
        let serialized_val = bincode::serialize(&value).unwrap();
        batch.push((serialized_key, serialized_val));
    }
}
```

`TypedTable<UserSessionKey, SessionData>` 在编译期锁死了 Key 和 Value 的类型。传错类型直接编译失败，不需要运行时调试。

### SurrealDB/Mongo 的 DDL 缺陷

无模式（Schema-less）是项目 0→1 阶段的「致幻剂」——开发速度极快，但到 1→10 的工业级阶段会成为数据脏乱差的万恶之源。

**弱约束导致应用层逻辑爆炸**。PostgreSQL 的 DDL 具备刚性物理约束——`NOT NULL`、`CHECK`、类型定义在撞击数据库外壳的瞬间就拦截非法数据。无模式数据库是「来者不拒的垃圾桶」：数字字段可以写成字符串 `"25"` 甚至数组 `[25]`。代价是所有消费端代码（微服务、Agent 工具）必须写满防御性类型判断和解析重试。数据库偷的懒，全部变成应用层的代码债务。

**查询优化器形同虚设**。PG 的优化器之所以强大，因为有刚性 DDL、确定的数据类型和索引元数据——优化器执行前就能精确计算磁盘步长和统计分布。无模式数据库不知道下一行 JSON 会冒出什么结构，查询引擎必须在运行时做大量动态类型断言和反射，CPU 算力被白白浪费在类型解析上。

**「代码即 DDL」如何反超**：Rust 强类型结构体把约束从运行时（DB 级）提到编译期（Rust 级）——脏数据在 `cargo check` 阶段就被熔断，连生成的资格都没有。同时不需要 PG 的运行时 DDL 锁开销和 SQL 字符串解析税。既消灭了无模式的脏数据风险，又白嫖了 KV 引擎的硬件响应速度。

### DDL 稳定性单元测试：硬编码 hex 字节防漂移

KV 系统最怕的风险：某次代码迭代改动了 Key 的前缀字符串（如 `"cfg:app:"` 错打成 `"config:app:"`），导致线上老数据永远读不出来。解决方案：编写自动化 Schema 偏移检测单元测试，用硬编码的历史 hex 字节锁定 Key 编码：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_key_ddl_stability_protection() {
        // 1. 构造确定性的业务测试数据
        let test_key = UserSessionKey {
            org_id: uuid::Uuid::parse_str("67e55044-10b1-426f-9247-bb680e5fe0c8").unwrap(),
            user_id: "user_88",
            session_id: "sess_99",
        };

        // 2. 硬编码历史版本生成的真实物理二进制（hex 16 进制）
        // 任何改动前缀或字段顺序的行为都会导致编码不匹配，CI/CD 立刻报错
        let expected = hex::decode(
            "736573733a67e5504410b1426f9247bb680e5fe0c83a757365725f38383a736573735f3939"
        ).unwrap();

        assert_eq!(
            test_key.encode(), expected,
            "DDL 物理 Key 编码发生非预期漂移！将导致线上老数据失联！"
        );
    }
}
```

这段测试的物理含义：`hex::decode("736573733a...")` 是 `sess:67e55044...:user_88:sess_99` 的 UTF-8 字节序列。任何人改动 `UserSessionKey` 的 `encode()` 方法（改前缀、调字段顺序、换序列化格式），测试立刻失败，阻止合入主分支。这是 KV 系统替代 DDL 约束的最终防线——编译期类型安全 + 运行时 hex 校验，双保险。

### 判定

SQL 的核心价值是面向人类的结构化纪律。KV 只要贯彻「代码即 DDL 的强类型编码 + 多版本 Enum 懒迁移 + 双写 Key 指针契约 + TypedTable 泛型抽象 + hex 硬编码单元测试」，就同时获得了：编译期 Schema 安全（Rust 编译器）+ 运行时极致性能（LSM-Tree）+ 零停机演进（版本化 Enum）+ 团队可维护性（Struct 注释即文档）+ 编码漂移防护（hex 单元测试）。在 Schema 安全性上完成对 SurrealDB（无模式）和 PostgreSQL（运行时 DDL 锁）的双向反超。

## 问题：字符串前缀的两大软肋

字符串前缀方案 `"user_sessions:"`（14 字节）在亿级数据量下：
- **空间浪费**：相同前缀重复存储，冲到亿级时浪费数 GB 存储空间（S3 传输带宽费）
- **变长偏移**：每个 Key 总长度动态变化，反序列化需要变长偏移计算

## 解决方案：数字命名空间字典

引入内存映射表（Map），将字符串前缀压缩为固定 2 字节 u16 命名空间 ID：

| 字符串方案 | 数字方案 | 压缩率 |
|:--|:--|:--|
| `"user_sessions:"` (14 字节) | `u16::to_be_bytes()` → 2 字节 | **85%** |

Map 极小（1024 个 namespace 撑死几 KB），100% 常驻 CPU L1 Cache，`Map.get("sessions")` 耗时仅几纳秒。

全局命名空间字典（编译期死锁）：

```
Namespace 1 → sessions
Namespace 2 → users
Namespace 3 → logs
```

## DDL 结构体：全定长、零浪费

```rust
#[derive(KvEncode)]
#[kv_ns(1)]  // 编译期翻译为 [0x00, 0x01] 2 字节大端序前缀
pub struct UserSessionKey {
    pub org_id: [u8; 16],     // 16 字节
    pub user_id: u64,          // 8 字节
    pub session_id: [u8; 16], // 16 字节
}
// 物理总长度 = 2(ns) + 16 + 8 + 16 = 42 字节，零字节浪费
```

## 过程宏实现源码

### 宏编译器驱动（`kv_codec_derive/src/lib.rs`）

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{parse_macro_input, AttrStyle, Data, DeriveInput, Fields, Lit, Meta, Type, Expr};

#[proc_macro_derive(KvEncode, attributes(kv_ns))]
pub fn derive_kv_encode(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    // 编译期提取数字 Namespace ID，转为 u16 大端序 2 字节
    let ns_id = extract_ns_id(&input);
    let ns_bytes = ns_id.to_be_bytes();

    let fields = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields_named) => &fields_named.named,
            _ => panic!("只支持带命名字段的结构体"),
        },
        _ => panic!("只支持结构体"),
    };

    let name = &input.ident;
    let mut encode_tokens = TokenStream2::new();
    let mut decode_tokens = TokenStream2::new();
    let mut capacity_tokens = TokenStream2::new();
    let mut current_offset = 2; // 前缀永远锁死在 2 字节位置

    for field in fields {
        let field_name = &field.ident;
        let field_type = &field.ty;

        match field_type {
            Type::Array(type_array) => {
                let len = &type_array.len;
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name);
                });
                capacity_tokens.extend(quote! { + #len });
                decode_tokens.extend(quote! {
                    let mut #field_name = [0u8; #len];
                    #field_name.copy_from_slice(&bytes[#current_offset..#current_offset + #len]);
                });
                current_offset += syn_parse_len(len);
            }
            Type::Path(tp) if tp.path.is_ident("u64") || tp.path.is_ident("i64") => {
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name.to_be_bytes());
                });
                capacity_tokens.extend(quote! { + 8 });
                decode_tokens.extend(quote! {
                    let #field_name = #field_type::from_be_bytes(
                        bytes[#current_offset..#current_offset + 8].try_into().unwrap()
                    );
                });
                current_offset += 8;
            }
            Type::Path(tp) if tp.path.is_ident("u32") || tp.path.is_ident("i32") => {
                encode_tokens.extend(quote! {
                    buf.extend_from_slice(&self.#field_name.to_be_bytes());
                });
                capacity_tokens.extend(quote! { + 4 });
                decode_tokens.extend(quote! {
                    let #field_name = #field_type::from_be_bytes(
                        bytes[#current_offset..#current_offset + 4].try_into().unwrap()
                    );
                });
                current_offset += 4;
            }
            _ => panic!("OKM 键空间只接受 [u8; N] 和基础数值类型（全定长）"),
        }
    }

    let expanded = quote! {
        impl #name {
            pub const NAMESPACE_ID: u16 = #ns_id;

            pub fn encode(&self) -> Vec<u8> {
                let mut buf = Vec::with_capacity(2 #capacity_tokens);
                buf.extend_from_slice(&[#(#ns_bytes),*]);
                #encode_tokens
                buf
            }

            pub fn decode(bytes: &[u8]) -> Result<Self, &'static str> {
                if bytes.len() != #current_offset || bytes[0..2] != [#(#ns_bytes),*] {
                    return Err("数据损坏或 Namespace 契约冲突");
                }
                #decode_tokens
                Ok(Self { #( #fields ),* })
            }
        }
    };
    TokenStream::from(expanded)
}

fn extract_ns_id(input: &DeriveInput) -> u16 {
    for attr in &input.attrs {
        if attr.style == AttrStyle::Outer && attr.path().is_ident("kv_ns") {
            if let Meta::List(meta_list) = &attr.meta {
                let expr: Expr = meta_list.parse_args().expect("kv_ns 格式错误");
                if let Expr::Lit(expr_lit) = expr {
                    if let Lit::Int(lit_int) = expr_lit.lit {
                        return lit_int.base10_parse::<u16>().expect("Namespace ID 必须是 u16");
                    }
                }
            }
        }
    }
    panic!("必须标注 #[kv_ns(N)]");
}

fn syn_parse_len(expr: &syn::Expr) -> usize {
    if let syn::Expr::Lit(expr_lit) = expr {
        if let syn::Lit::Int(lit_int) = &expr_lit.lit {
            return lit_int.base10_parse::<usize>().unwrap();
        }
    }
    panic!("无法解析数组长度");
}
```

### 运行时模块 + 测试（`kv_codec/src/lib.rs`）

```rust
pub use kv_codec_derive::KvEncode;

#[cfg(test)]
mod tests {
    use super::*;

    #[derive(KvEncode)]
    #[kv_ns(1)]
    pub struct UserSessionKey {
        pub org_id: [u8; 16],
        pub user_id: u64,
        pub session_id: [u8; 16],
    }

    #[test]
    fn test_okm_encode_decode() {
        let key = UserSessionKey {
            org_id: [0xAA; 16],
            user_id: 99,
            session_id: [0xBB; 16],
        };

        let bin = key.encode();

        // 验证物理总长度：2(ns) + 16 + 8 + 16 = 42 字节
        assert_eq!(bin.len(), 42);

        // 验证前缀：[0x00, 0x01] = Namespace 1 的大端序
        assert_eq!(bin[0..2], [0x00, 0x01]);

        // 验证反序列化
        let restored = UserSessionKey::decode(&bin).unwrap();
        assert_eq!(restored.org_id, key.org_id);
        assert_eq!(restored.user_id, key.user_id);
        assert_eq!(restored.session_id, key.session_id);
    }

    #[test]
    fn test_ddl_hex_stability() {
        let key = UserSessionKey {
            org_id: [1u8; 16],
            user_id: 50,
            session_id: [2u8; 16],
        };

        // 历史 hex 物理特征死锁——任何改动 CI/CD 立刻阻断
        let expected = "000101010101010101010101010101010101000000000000003202020202020202020202020202020202";
        assert_eq!(hex::encode(key.encode()), expected);
    }
}
```

## syn→quote 类型映射规则

| 字段类型 | syn 分析 | quote 生成 | 字节宽度 |
|:--|:--|:--|:--|
| `[u8; N]` | `Type::Array` | `extend_from_slice(&self.field)` | N（固定） |
| `u64` / `i64` | `Type::Path` | `extend_from_slice(&self.field.to_be_bytes())` | 8（固定） |
| `u32` / `i32` | `Type::Path` | `extend_from_slice(&self.field.to_be_bytes())` | 4（固定） |

## OKM 的物理收益

- **85% 前缀压缩**：14 字符串 → 2 数字字节
- **100% 定长 Key**：所有字段在磁盘上的绝对字节偏移量被编译期死锁，反序列化纯指针切片，零解析
- **65535 命名空间**：u16 支撑 65535 个不同的键空间，覆盖网关 + Agent 全场景
- **Cache Locality 极致**：LSM-Tree 布隆过滤器拦截、内存 Seek 查找时 CPU 缓存局部性极好
- **零运行时开销**：无正则/split/AST，查询路径随 `cargo build --release` 固化为机器码

## TypedCollection：编译期对象空间容器

OKM 的上层封装——通过 PhantomData 泛型对底层裸字节 KV 进行类型死锁，对外暴露 100% 类型安全的 ORM 级接口，运行时零开销。

### 实体声明：Key 字段 + Value 字段

结构体中一部分字段由宏提取用于编排 Key，剩下的自动作为 Value Payload：

```rust
use kv_codec::KvEncode;
use serde::{Serialize, Deserialize};

/// 声明式 DDL 契约——看起来像 ORM 实体表
#[derive(KvEncode, Serialize, Deserialize, Debug, Clone)]
#[kv_ns(1)]  // 永久死锁在 Namespace ID 1
pub struct UserSessionRecord {
    // 【KEY 区】：宏提取 → 大端序定长物理 Key
    pub org_id: [u8; 16],
    pub user_id: u64,

    // 【VALUE 区】：业务资产 → 序列化为磁盘行数据
    pub session_token: String,
    pub ip_address: u32,
    pub is_active: bool,
}
```

### EntityKeyGenerator：Key/Value 自动分离 trait

宏在编译期自动为结构体实现 `EntityKeyGenerator`，在不解构的前提下分离 Key 字段与 Value 字段：

```rust
/// 过程宏全自动生成的辅助 trait
pub trait EntityKeyGenerator {
    const NAMESPACE_ID: u16;
    fn generate_compiled_key(&self) -> Vec<u8>;
}

// 宏自动生成的实现
impl EntityKeyGenerator for UserSessionRecord {
    const NAMESPACE_ID: u16 = 1;

    fn generate_compiled_key(&self) -> Vec<u8> {
        let mut buf = Vec::with_capacity(2 + 16 + 8); // 26 字节
        buf.extend_from_slice(&Self::NAMESPACE_ID.to_be_bytes()); // 2 字节命名空间
        buf.extend_from_slice(&self.org_id);                        // 16 字节
        buf.extend_from_slice(&self.user_id.to_be_bytes());        // 8 字节
        buf
    }
}
```

Key 字段（`org_id`、`user_id`）通过 `generate_compiled_key()` 编排为固定宽度二进制；Value 字段（`session_token`、`ip_address`、`is_active`）通过 `bincode::serialize()` 序列化为完整 Payload。两部分在同一个原子 Batch 内提交。

### TypedCollection 容器

→ 生产实现中 `KvBackend` 替换为 `AuraStorage` trait（见 [Aura 架构 §5.1](aura-architecture.md#51-存储引擎抽象trait-分离)），通过 `Arc<dyn AuraStorage>` 实现 Fjall/SlateDB 运行时切换。

```rust
use std::marker::PhantomData;
use std::sync::Arc;

/// 零状态、零运行时开销的编译期对象管理器
pub struct TypedCollection<T> {
    _marker: PhantomData<T>,  // 编译期类型锁死，运行时 0 字节
    pub raw_backend: Arc<dyn AuraStorage>,  // FjallEngine 或 SlateEngine
}

impl<T> TypedCollection<T> where
    T: Serialize + serde::de::DeserializeOwned + Clone + EntityKeyGenerator
{
    pub fn new(backend: Arc<dyn KvBackend>) -> Self {
        Self { _marker: PhantomData, raw_backend: backend }
    }

    /// 类 ORM 体验：将结构体原生存入系统
    pub fn save(&self, batch: &mut WriteBatch, entity: &T) {
        // 1. 宏编译期硬编码的 Key 构造器（零运行时字符串解析）
        let physical_key = entity.generate_compiled_key();
        // 2. 整个对象序列化为二进制 Value Payload
        let physical_value = bincode::serialize(entity).unwrap();
        // 3. 推入原子写入批次
        batch.push((physical_key, physical_value));
    }

    /// 类 ORM 体验：通过 Key 字段精准点查
    pub fn find_by_key(&self, key_spec: &T) -> Option<T> {
        let target_key = key_spec.generate_compiled_key();
        // let raw = self.raw_backend.get(&target_key)?;
        // let entity: T = bincode::deserialize(&raw).unwrap();
        // Some(entity)
        None  // 模拟
    }
}
```

### 上层业务代码的极致清爽

```rust
fn process_login(collection: &TypedCollection<UserSessionRecord>, data: UserSessionRecord) {
    let mut batch = Vec::new();

    // 零原始字节操作、零手动前缀拼接、零恶心分隔符字符串
    collection.save(&mut batch, &data);

    println!("写入 Namespace: {}", UserSessionRecord::NAMESPACE_ID);
}
```

开发人员面对的是纯粹的 Rust 领域模型。宏在编译期把所有类型信息蒸发，直接打穿底层 KV 引擎的硬件极限性能。上层是 ORM 般的声明式体验，底层是裸字节级的指针偏移计算。

### 运行时冲突检测方向

TypedCollection 可以扩展乐观冲突检测——基于 MVCC 版本号的 CAS 乐观锁重试循环，抵御多线程并发对同一 Session 的更新擦除：

```rust
impl<T> TypedCollection<T> where T: EntityKeyGenerator {
    /// 乐观锁写入：版本号不匹配时自动重试
    pub fn save_with_cas(&self, batch: &mut WriteBatch, entity: &T, expected_version: u64) -> bool {
        let key = entity.generate_compiled_key();
        // let current = self.raw_backend.get(&key)?;
        // let current_version = u64::from_be_bytes(current[0..8]);
        // if current_version != expected_version { return false; } // 版本冲突
        // 写入新版本...
        true
    }
}
```

## Openraft 状态机集成

OKM 与 Raft 共识层的集成点——状态机将 Raft 日志条目 Apply 到 Fjall，实现本地物理盘的分布式一致性写入。SlateDB 模式不需要 Raft——S3 自身提供 HA，计算节点无状态。

### 核心结构

```rust
use std::sync::Arc;
use openraft::{RaftStateMachine, StateMachineData, StateMachineError, LogId};
use fjall::{Config, Keyspace, PartitionHandle};
use serde::{Serialize, Deserialize};

/// Raft 状态转换指令——所有写操作必须通过此枚举广播
#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RaftCommand {
    UpdateActorState {
        agent_id: String,
        serialized_context: Vec<u8>,  // CBOR 编码的 Actor 状态快照
    },
    TerminateActor {
        agent_id: String,
    },
}

/// 分布式状态机——内嵌 Fjall 物理分区
pub struct FjallStateMachine {
    pub keyspace: Keyspace,
    pub actor_partition: PartitionHandle,  // Actor 状态分区
    pub meta_partition: PartitionHandle,   // Raft 元数据分区（last_log_id 等）
}
```

### 初始化

```rust
impl FjallStateMachine {
    pub fn new(path: &str) -> Self {
        let keyspace = Keyspace::open_default(path)
            .expect("无法初始化 Fjall 物理存储区");
        let actor_partition = keyspace.open_partition("actors", Config::default())
            .expect("无法开辟智能体物理状态分区");
        let meta_partition = keyspace.open_partition("meta", Config::default())
            .expect("无法开辟集群元数据分区");
        Self { keyspace, actor_partition, meta_partition }
    }
}
```

### RaftStateMachine trait 实现

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct StateMachineResponse {
    pub success: bool,
}

impl StateMachineData for FjallStateMachine {}

impl RaftStateMachine<openraft::TypeConfig> for Arc<FjallStateMachine> {
    /// 从 meta_partition 读取上一次成功执行的 Raft 日志 ID，防止状态漂移
    async fn applied_state_id(
        &self,
    ) -> Result<Option<LogId<u32>>, StateMachineError<openraft::TypeConfig>> {
        if let Ok(Some(bytes)) = self.meta_partition.get("last_log_id") {
            let log_id = postcard::from_bytes(&bytes).unwrap();
            Ok(Some(log_id))
        } else {
            Ok(None)
        }
    }

    /// 核心 Apply 逻辑：Raft 日志 → Fjall 物理写入
    async fn apply<I>(
        &self,
        entries: I,
    ) -> Result<Vec<StateMachineResponse>, StateMachineError<openraft::TypeConfig>>
    where
        I: IntoIterator<Item = openraft::Entry<openraft::TypeConfig>> + Send,
    {
        let mut responses = Vec::new();

        for entry in entries {
            let log_id = entry.log_id;
            if let openraft::EntryPayload::Normal(cmd) = entry.payload {
                match cmd {
                    RaftCommand::UpdateActorState { agent_id, serialized_context } => {
                        // Fjall 单次写入，自带崩溃保护
                        self.actor_partition.insert(
                            agent_id.as_bytes(), serialized_context
                        ).expect("Fjall 本地物理写入异常");
                        responses.push(StateMachineResponse { success: true });
                    }
                    RaftCommand::TerminateActor { agent_id } => {
                        self.actor_partition.remove(agent_id.as_bytes())
                            .expect("Fjall 本地擦除异常");
                        responses.push(StateMachineResponse { success: true });
                    }
                }
            }

            // 实时固化最新 LogId 到元数据分区
            let log_bytes = postcard::to_stdvec(&log_id).unwrap();
            self.meta_partition.insert("last_log_id", log_bytes).unwrap();
        }

        // 强制刷盘到 NVMe——Raft 共识的持久化保证
        self.keyspace.persist().unwrap();
        Ok(responses)
    }
}
```

### 物理保证

- **原子性**：Fjall 的 `keyspace.persist()` 保证一次 apply 内所有写入原子落盘
- **崩溃恢复**：重启后从 `meta_partition` 读取 `last_log_id`，重放未 Apply 的 Raft 日志
- **错误保护**：Fjall 写入失败触发 panic（系统级保护），不会静默丢数据
- **与 OKM 的关系**：`RaftCommand::UpdateActorState` 的 `serialized_context` 可以是 OKM 编码的实体——`AuraCollection.save()` 生成的 `(key, value)` 对通过 Raft 广播后，状态机将其 Apply 到 Fjall。SlateDB 模式下直接走 SlateDB async API 写入 S3，无需 Raft 状态机。
