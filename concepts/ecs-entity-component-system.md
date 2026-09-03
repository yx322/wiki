---
title: ECS 实体组件系统
created: 2026-08-12
updated: 2026-08-12
type: concept
tags: [ecs, bevy, game-engine, rust]
sources: [raw/articles/entity-component-system.md]
confidence: high
---

# ECS 实体组件系统

ECS（Entity-Component-System）是 Rust 对 OO 的回应——一种数据+逻辑组织范式，核心思想是**标签（一物多标签 = 倒排索引）**替代**树（单主体命名 = class 继承）**。

## 范式本质：标签 vs 树

| 维度 | OO（class 继承）| ECS |
|------|----------------|-----|
| 结构 | **树**：层层向上收敛到唯一根类 | **标签**：一物多标签，自由组合 |
| 身份来源 | class 命名（"它是谁"）| 组件组合（"它有哪些方面"）|
| 数据与逻辑 | 绑定在一起（方法挂在对象上）| **分离**（数据在组件，逻辑在系统）|
| 类型变更 | 固定（class 定义后不可变）| 运行时增删组件即可改身份 |
| 多态类型 | 包含多态（子类型/里氏替换）| 参数多态（`T: A + B`）|

**ECS 是 Rust 对 OO 的回应。** Rust 没有 class，无类+标签化要落到"大量异质实体持续演化、彼此交互"的领域，就长出了 ECS。

## 三层模型（数据库心智模型）

ECS 的心智模型不是对象树，而是**扁平的标签库，像数据库**：

| ECS 角色 | 数据库类比 | 说明 |
|----------|-----------|------|
| **Entity** | 行键 | 纯标识（句柄），本身无内容 |
| **Component** | 可查询的维度（列/标签）| 存数据或作身份标记 |
| **System** | 查询 + 批操作 | 按组件筛选实体，对结果集统一执行 |

**实体是什么，完全由它身上的组件组合决定**，不来自任何类定义。

## 收益画像

ECS 的四项收益只在**同一个画像**下才划算：

| 收益 | 前提条件 |
|------|---------|
| **缓存局部性** | 同类型组件连续存储（SoA），系统批量遍历时走内存顺序访问 |
| **无状态同步** | 状态躺在组件里，系统 Query 直读，无需 getter/setter |
| **运行时改身份** | 同一句柄增删组件即改类型（活的→尸体）|
| **系统可并行** | 调度器按读写依赖，把无共享状态的系统并行跑 |

**这个画像 = 游戏/模拟**：实体数量大、性能敏感/实时、实体类型运行时可变。

离开这个画像，收益消失、成本显形：
- 实体少 → 缓存局部性无所谓
- 类型稳定 → 运行时改身份用不上
- 非实时 → 没理由为并行付调度开销

## 恒定代价

| 代价 | 说明 |
|------|------|
| **间接性** | 实体是匿名行键，要知道"它是什么"必须查组件 |
| **失去封装** | 行为散在系统里，"方法属于对象"的局部性没了 |
| **查询开销** | 每次访问都过组件存储，不如 OO 直接字段访问 |

**结论**：业务 CRUD、低实体数、类型不变——OO 或普通数据结构更简单，ECS 不是普遍替代。

## Bevy 核心 API（0.19）

**心智模型**：Bevy 把整个游戏做成一个 App——App 里有一个 ECS World（所有实体/组件/资源都住在里面），外加控制"系统何时运行"的 Schedule。**数据在 World 里、逻辑在 System 里、时机在 Schedule 里**。

```rust
fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .init_resource::<Score>()
        .add_systems(Startup, setup)
        .add_systems(Update, (move_player, update_score))
        .run();
}

fn move_player(q: Query<&mut Transform, With<Player>>) {
    for mut t in &q { /* 每帧移动 */ }
}
```

**关键**：你从不写"对象方法"，你写系统。每个系统是一个函数，声明自己要什么（Query 读哪些组件），然后被调度到某个 Schedule 里反复跑。

## 与 trait 的关系

ECS 的 `component` 与 `trait`/tag/label 是同一类东西：都是"挂在标识上的标签"。区别只是 ECS 的组件比纯标签多承载了数据（`Health(3)`、`Transform`），是"标签 + 数据"。

系统按组件筛选实体（Query），本质就是倒排索引求交集——一个实体可同时带 Player/Health/Transform 多组标签，正是"一物多标签"。

## 参见
- [[modern-language-design]] — 现代语言设计（标签 vs 树的完整论证）
- [[ecs-benefits-profile]] — ECS 收益画像分析

^[raw/articles/entity-component-system.md]
