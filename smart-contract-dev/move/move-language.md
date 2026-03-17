# Move 语言核心

> **标签：** move、资源模型、能力、类型系统

Move 是专为区块链设计的编程语言，最大创新是**资源类型（Resource Types）**：数字资产作为一等公民，从语言层面保证不可凭空创建或销毁。

---

## 1. 核心哲学：资源安全

与 Solidity 不同，Move 从**类型系统**层面防止双花：

```
Solidity：
  mapping(address => uint256) balances;
  // 余额只是一个数字，可以随意修改
  // 安全靠开发者正确编写检查逻辑

Move：
  struct Coin has store { value: u64 }
  // Coin 是资源，不能复制（无 copy），不能销毁（无 drop）
  // 必须显式存储或转移，不能凭空消失
```

---

## 2. 四种能力（Abilities）

每个 Move 类型可具备 0-4 种能力：

| 能力 | 说明 | 缺省行为 |
|------|------|---------|
| `copy` | 值可被复制 | 无法复制（必须移动） |
| `drop` | 值可被丢弃 | 必须显式使用（存储或转移） |
| `store` | 值可存入全局存储 | 不能作为字段存储 |
| `key` | 值可作为全局存储的顶层键 | 不能作为顶层资源 |

```move
// 普通值类型（可复制、可丢弃）
struct Point has copy, drop {
    x: u64,
    y: u64,
}

// 资源类型（不可复制、不可丢弃）
struct Coin has key, store {
    value: u64,
}
// Coin 实例必须被存入账户或传给其他函数，不能凭空消失
```

---

## 3. Move 语法基础

### 模块（Module）

```move
module my_addr::counter {
    use std::signer;
    use aptos_framework::account;

    struct Counter has key {
        value: u64,
    }

    public fun initialize(account: &signer) {
        move_to(account, Counter { value: 0 });
    }

    public fun increment(account: &signer) acquires Counter {
        let counter = borrow_global_mut<Counter>(signer::address_of(account));
        counter.value = counter.value + 1;
    }

    #[view]
    public fun get_value(addr: address): u64 acquires Counter {
        borrow_global<Counter>(addr).value
    }
}
```

### 引用与借用

```move
// 不可变引用（读）
let ref: &Counter = borrow_global<Counter>(addr);
let value = ref.value; // 读取

// 可变引用（写）
let ref_mut: &mut Counter = borrow_global_mut<Counter>(addr);
ref_mut.value = ref_mut.value + 1; // 修改

// 函数参数引用
public fun add(x: &mut u64, y: u64) {
    *x = *x + y; // 解引用修改
}
```

---

## 4. 全局存储操作

Move 使用内置原语操作账户下的全局存储：

| 操作 | 说明 |
|------|------|
| `move_to(signer, resource)` | 将资源存入签名者的账户 |
| `move_from<T>(addr)` | 从账户取出资源（销毁存储，返回值） |
| `borrow_global<T>(addr)` | 不可变借用账户下的资源 |
| `borrow_global_mut<T>(addr)` | 可变借用账户下的资源 |
| `exists<T>(addr)` | 检查账户下是否存在资源 |

```move
// 完整的资源生命周期
public fun create(account: &signer, value: u64) {
    assert!(!exists<MyResource>(signer::address_of(account)), 1); // 检查不重复
    move_to(account, MyResource { value });
}

public fun destroy(account: &signer): u64 acquires MyResource {
    let MyResource { value } = move_from<MyResource>(signer::address_of(account));
    value // 返回内部值（资源被销毁）
}
```

---

## 5. 向量（Vector）

Move 标准库提供向量操作：

```move
use std::vector;

let v: vector<u64> = vector::empty();
vector::push_back(&mut v, 1);
vector::push_back(&mut v, 2);
let len = vector::length(&v); // 2
let first = *vector::borrow(&v, 0); // 1
vector::pop_back(&mut v); // 移除并返回最后一个
```

---

## 6. 泛型

```move
// 泛型结构
struct Wrapper<T> has copy, drop {
    value: T,
}

// 泛型函数
public fun wrap<T: copy + drop>(value: T): Wrapper<T> {
    Wrapper { value }
}

// 泛型约束：T 必须有 store 能力
struct Container<T: store> has key {
    items: vector<T>,
}
```

---

## 7. Move vs Solidity 关键差异

| 维度 | Move | Solidity |
|------|------|---------|
| 资产安全 | 类型系统保证（能力约束） | 开发者自行保证（逻辑检查） |
| 重入防护 | 天然防护（资源移动语义） | 需要 CEI 或 nonReentrant |
| 全局状态 | 账户下的资源（分散） | 合约内 mapping（集中） |
| 继承 | 无继承（模块化组合） | 支持继承 |
| 升级 | 可选（Sui 和 Aptos 支持） | 需代理模式 |

---

## 参考资源

- [Move Book](https://move-book.com/)（最佳入门书）
- [Move 语言规范](https://github.com/move-language/move/tree/main/language/documentation/spec)
- [Sui Move vs Aptos Move 差异](https://docs.sui.io/concepts/sui-move-concepts)
