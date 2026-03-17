# Aptos Move 开发

> **标签：** aptos、move、资源账户、fungible-asset、aptos-framework

Aptos 使用与 Sui 类似但有重要区别的 Move 版本，采用**账户（Account）为中心**的模型，状态存储在账户下的资源（Resources）中。

---

## 1. Aptos 账户模型

```
Aptos 账户
├── 地址（address）
├── 认证密钥（authentication_key）
├── 序列号（sequence_number，防重放）
├── 资源（Resources）
│   ├── 0x1::coin::CoinStore<AptosCoin>
│   ├── 0x1::account::Account
│   └── 自定义资源...
└── 模块（Modules）—— 部署的代码
```

与 Sui 不同：Aptos 中状态是账户下的资源，而非全局对象集合。

---

## 2. Aptos Move 基础

### 模块与资源

```move
module my_addr::counter {
    use std::signer;
    use aptos_framework::account;
    use aptos_std::table::{Self, Table};

    struct Counter has key {
        value: u64,
    }

    // 初始化：在签名者账户下存储资源
    public entry fun initialize(account: &signer) {
        let addr = signer::address_of(account);
        assert!(!exists<Counter>(addr), 1);
        move_to(account, Counter { value: 0 });
    }

    public entry fun increment(account: &signer) acquires Counter {
        let addr = signer::address_of(account);
        let counter = borrow_global_mut<Counter>(addr);
        counter.value = counter.value + 1;
    }

    #[view]
    public fun get_count(addr: address): u64 acquires Counter {
        borrow_global<Counter>(addr).value
    }
}
```

### `entry` 函数

Aptos 用 `entry` 关键字标记可被直接调用的函数（类似 Solidity 的 `external`）：

```move
// entry 函数：可被交易直接调用
public entry fun transfer(from: &signer, to: address, amount: u64) acquires CoinStore {
    // ...
}

// 非 entry 函数：只能被其他 Move 函数调用
public fun calculate_fee(amount: u64): u64 {
    amount / 100
}
```

---

## 3. Aptos Token 标准

### Coin（旧标准）

```move
use aptos_framework::coin::{Self, Coin};
use aptos_framework::managed_coin;

// 定义代币
struct MyToken has key {}

// 初始化（一次性）
public entry fun init_coin(account: &signer) {
    managed_coin::initialize<MyToken>(
        account,
        b"My Token",
        b"MTK",
        8,    // decimals
        false, // monitor_supply
    );
}

// 铸造
public entry fun mint(account: &signer, to: address, amount: u64) {
    managed_coin::mint<MyToken>(account, to, amount);
}
```

### Fungible Asset（新标准，推荐）

```move
use aptos_framework::fungible_asset::{Self, MintRef, TransferRef, BurnRef, Metadata};
use aptos_framework::primary_fungible_store;
use aptos_framework::object::{Self, ConstructorRef};

// 创建 Fungible Asset
public fun init_fa(constructor_ref: &ConstructorRef) {
    primary_fungible_store::create_primary_store_enabled_fungible_asset(
        constructor_ref,
        option::none(),     // supply（无上限）
        utf8(b"My Token"),  // name
        utf8(b"MTK"),       // symbol
        8,                  // decimals
        utf8(b""),          // icon uri
        utf8(b""),          // project uri
    );
}
```

---

## 4. 资源账户（Resource Account）

资源账户是**程序控制的账户**（类似 Solana PDA），用于让程序持有资源：

```move
use aptos_framework::resource_account;
use aptos_framework::account;

// 创建资源账户（由部署者账户 + seed 派生）
resource_account::create_resource_account(
    &deployer_signer,
    b"my_vault_seed",
    vector::empty(),  // auth key 前缀
);

// 获取资源账户签名者（程序可用来代表资源账户签名）
let (resource_signer, signer_cap) = account::create_resource_account(
    &deployer_signer,
    b"my_vault_seed",
);

// 存储签名者能力（后续操作用）
move_to(&resource_signer, VaultConfig {
    signer_cap,
    // ...
});

// 使用签名者能力（让资源账户执行操作）
let resource_signer = account::create_signer_with_capability(&config.signer_cap);
coin::transfer<AptosCoin>(&resource_signer, recipient, amount);
```

---

## 5. 事件（Events）

```move
use aptos_framework::event;

#[event]
struct TransferEvent has drop, store {
    from: address,
    to: address,
    amount: u64,
}

// 发出事件
event::emit(TransferEvent { from, to, amount });
```

---

## 6. Aptos 开发工具链

```bash
# 安装 Aptos CLI
curl -fsSL "https://aptos.dev/scripts/install_cli.py" | python3
# 或
brew install aptos

# 初始化项目
aptos move init --name my_project

# 编译
aptos move compile

# 测试
aptos move test

# 发布（devnet）
aptos move publish --named-addresses my_addr=$(aptos config show-profiles | grep account | head -1 | awk '{print $2}')

# 调用函数
aptos move run \
  --function-id 'MY_ADDR::counter::initialize' \
  --profile devnet
```

### Move.toml（Aptos）

```toml
[package]
name = "my_project"
version = "1.0.0"

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[addresses]
my_addr = "_"  # 编译时替换
aptos_std = "0x1"
```

---

## 7. Sui vs Aptos 开发对比

| 维度 | Sui | Aptos |
|------|-----|-------|
| 状态模型 | 对象（全局 Object 集合） | 资源（账户下的 Resource） |
| 程序控制账户 | 无需额外机制（对象可由程序拥有） | 资源账户（resource_account） |
| 可组合性 | PTB（可编程交易块） | 多重签名 + 序列化调用 |
| 代币标准 | Coin<T> / Fungible Asset | Coin（旧）/ Fungible Asset（新） |
| 并行执行 | 对象级（无冲突对象并行） | Block-STM（乐观并发） |

---

## 参考资源

- [Aptos 开发者文档](https://aptos.dev/)
- [Move Prover（形式化验证）](https://aptos.dev/move/prover/intro)
- [Aptos TypeScript SDK](https://aptos.dev/sdks/ts-sdk/)
- [Aptos Fungible Asset 标准](https://aptos.dev/standards/fungible-asset)
