---
title: Rust 多线程入门：Arc + Mutex + thread::spawn
created: 2026-10-28
updated: 2026-10-28
type: concept
tags: [rust, architecture]
sources: []
confidence: high
---

# Rust 多线程入门：Arc + Mutex + thread::spawn

10 个线程同时给一个计数器 `+1`，最终结果必须是 10。难点：Rust 所有权默认独占，一个数据不能同时被多个线程拥有，所以需要 `Arc` + `Mutex` 组合 —— [[modern-language-design]] 中"复杂度守恒"的典型体现：语言把并发安全的复杂度显式交给了类型系统。

## 完整示例

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));   // Arc 套 Mutex 套数据
    let mut handles = vec![];

    for i in 0..10 {
        let counter = Arc::clone(&counter);  // 克隆引用（不是数据）
        let handle = thread::spawn(move || { // move 把所有权搬进线程
            let mut num = counter.lock().unwrap(); // 拿锁 → 拆 Result
            *num += 1;                          // 解引用修改
            println!("线程 {} 累加完成，当前值: {}", i, *num);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();              // 等所有线程收工
    }

    println!("最终计数: {}", *counter.lock().unwrap());
}
```

## 为什么 Arc 和 Mutex 必须成对出现

| 只用其一 | 结果 |
|---|---|
| 只用 `Mutex` | 能互斥访问，但没有所有权共享能力，无法"交给"10 个线程 |
| 只用 `Arc` | 能共享所有权，但不阻止并发写 → 数据竞争，结果 < 10 |
| `Arc<Mutex<T>>` | Arc 负责"都摸得到"，Mutex 负责"同时只有一人动手" |

**类比**：Arc 是把同一个房间的钥匙复印 10 份发出去；Mutex 是门口的门锁——谁进去谁锁门，出来才还钥匙。

## Arc::clone：复制的是指针，不是数据

```
栈（每个变量一个）              堆（只有一份）
┌──────────────┐
│ 克隆的 counter │───▶│  强引用计数: 2          │
└──────────────┘      │  Mutex<i32> = 0        │
┌──────────────┐      │  （真正的数据）          │
│ 主线程 counter │───▶│                        │
└──────────────┘      └────────────────────────┘
```

- `Arc::clone(&counter)` ≡ `counter.clone()`，只复制 8 字节指针 + 原子计数 +1，与数据大小无关
- 强引用计数归零时数据才真正释放；计数器本身用**原子操作**维护，多线程同时 clone/drop 也不会乱（这就是 Arc 名字里 **A**tomic 的含义；普通 `Rc` 非原子，被禁止跨线程）
- 同名 `let counter = Arc::clone(&counter)` 是**变量遮蔽**：循环里 counter 永远指"本次克隆的副本"
- 必须克隆的原因：`move` 会把变量所有权搬进第一个线程，不克隆则第二轮循环 `use of moved value` 编译错误

## 循环体逐次执行的流程

每次迭代就三步：**克隆 Arc → spawn 线程 → 存句柄**。

- `thread::spawn(闭包)` 立刻返回 `JoinHandle`，不等线程干完——主循环飞快转完 10 次，10 个线程同时在后台跑
- `move ||` 把克隆的 `counter` 和 `i` 搬进闭包（线程可能活得比外层代码久，借用会被编译器拒绝）
- `handles.push(handle)` 攒收据，循环后 `join()` 逐个等待，保证读结果时所有累加已完成
- 抢锁顺序随机 → 线程编号乱序打印，但"当前值"一定是 1..10 递增（+1 排队进行，绝不漏加）

## .lock().unwrap() 拆解

```rust
let mut num = counter.lock().unwrap();
```

- `counter.lock()`：Mutex 的**唯一入口**。锁被占用则阻塞等待；返回 `Result<MutexGuard, PoisonError>`（锁中毒机制：持锁线程 panic 过 → 锁标记中毒 → 后续 lock 得 Err）
- `.unwrap()`：拆 Result 盒子——Ok 取出 MutexGuard，Err 直接 panic。Rust 无异常/null，可能失败的返回值都包在 `Result<T, E>` 里，编译器强制你处理
- `let mut num` 必须 mut：要通过 `DerefMut` 可变解引用执行 `*num += 1`
- MutexGuard 离开作用域**自动释放锁**，无需手动 unlock；且 Guard 是 `!Send`，拿着锁跨线程会被编译器拦截
- `unwrap` 非唯一选择：`expect`（带自定义报错）、`match`、`unwrap_or_else`、`?`（向上传递）皆可，教程用 unwrap 纯图省事

## 语法速记

| 符号 | 含义 | 例子 |
|---|---|---|
| `::` | 模块/类型的关联函数 | `thread::spawn`、`Arc::new`、`Mutex::new` |
| `.` | 实例方法 | `counter.lock()`、`handle.join()` |
| `move \|\|` | 转移所有权进闭包 | `thread::spawn(move \|\| {...})` |
| `*x` | 解引用 | `*num += 1` |
| `0..10` | Range，含 0 不含 10 | `for i in 0..10` |
| `vec![]` | 可增长数组字面量 | `let mut handles = vec![]` |

## 常见坑

| 坑 | 原因 |
|---|---|
| `let num = ...` 少写 `mut` | 修改数据需要可变绑定 |
| 忘记 `move` | 线程生命周期长于借用，编译器拒绝 |
| 忘记 clone Arc | 所有权被第一个线程搬走，循环第二轮报错 |
| 在锁内做耗时操作 | 其他线程全部阻塞，性能变差 |
| 克隆 Mutex 或内部数据 | 破坏"共享同一份"的语义，各改各的账 |

## 相关

- Rust 生态中的实际应用：[[fjall]]、[[slatedb]]、[[polars]]、[[openraft]] 均大量使用 Arc/Mutex 做并发共享
- 语言设计层面：[[modern-language-design]] —— Rust 用所有权系统把内存/并发安全前移到编译期

## Sources

- 对话整理（2026-10-28）：Rust 多线程代码逐行讲解 + Arc::clone 深度剖析 + for 循环逐步执行 + unwrap/Result 机制
