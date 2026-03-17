# Sui Move 开发

> **标签：** sui、move、对象模型、ptb、dynamic-fields

Sui 基于 Move 语言，但对原版 Move 做了重要改造：**以对象（Object）为核心**，而非账户地址。

---

## 1. Sui 对象模型

```
Sui 全局状态 = 对象集合（而非账户映射）

Object：
├── id: UID                    // 唯一标识符
├── version: u64               // 每次修改自增
├── digest: bytes              // 内容哈希
└── owner:
    ├── Address(addr)          // 由某地址拥有（常见）
    ├── Shared { initial_shared_version } // 共享对象（需共识）
    ├── Immutable              // 不可变对象
    └── Object(parent_id)      // 对象内嵌（动态字段）
```

---

## 2. Sui Move 基础

### 对象定义

```move
module my_package::nft {
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};
    use sui::transfer;
    use std::string::String;

    struct NFT has key, store {
        id: UID,         // 每个 Sui 对象必须有 UID
        name: String,
        image_url: String,
    }

    // 创建 NFT（用 key 能力 = 顶层对象）
    public fun mint(name: String, image_url: String, ctx: &mut TxContext): NFT {
        NFT {
            id: object::new(ctx),  // 生成唯一 ID
            name,
            image_url,
        }
    }

    // 转移 NFT
    public entry fun transfer_nft(nft: NFT, recipient: address) {
        transfer::public_transfer(nft, recipient);
    }

    // 销毁 NFT
    public entry fun burn(nft: NFT) {
        let NFT { id, name: _, image_url: _ } = nft;
        object::delete(id);
    }
}
```

### 共享对象（需要共识）

```move
// 共享对象：任何人可以修改，需要全局共识排序
struct Counter has key {
    id: UID,
    value: u64,
}

public fun create_counter(ctx: &mut TxContext) {
    let counter = Counter { id: object::new(ctx), value: 0 };
    transfer::share_object(counter); // 变为共享对象
}

public entry fun increment(counter: &mut Counter) {
    counter.value = counter.value + 1;
}
```

---

## 3. 可编程交易块（PTB）

PTB 是 Sui 最强大的特性：一个交易可以链式调用多个函数，前步输出直接作为后步输入：

```move
// 链上部分：定义可组合函数
public fun split_coin(coin: &mut Coin<SUI>, amount: u64, ctx: &mut TxContext): Coin<SUI> {
    coin::split(coin, amount, ctx)
}

public fun deposit_to_vault(coin: Coin<SUI>, vault: &mut Vault) {
    vault.balance = vault.balance + coin::value(&coin);
    coin::put(&mut vault.funds, coin);
}
```

```typescript
// 客户端 PTB：组合多个操作为单笔原子交易
import { Transaction } from '@mysten/sui/transactions'

const tx = new Transaction()

// 1. 分割 coin
const [coin] = tx.splitCoins(tx.gas, [1_000_000_000n])

// 2. 将 coin 存入 vault（复用上一步的 coin）
tx.moveCall({
    target: 'PACKAGE::vault::deposit_to_vault',
    arguments: [coin, tx.object(VAULT_ID)],
})

// 3. 铸造 NFT
const [nft] = tx.moveCall({
    target: 'PACKAGE::nft::mint',
    arguments: [tx.pure.string('My NFT'), tx.pure.string('https://...')],
})

// 4. 转移 NFT 给另一个地址
tx.transferObjects([nft], recipientAddress)

// 以上 4 步在一个原子交易中执行
const result = await client.signAndExecuteTransaction({ signer, transaction: tx })
```

---

## 4. 动态字段（Dynamic Fields）

允许在运行时向对象添加字段（突破 Sui Move 的静态结构限制）：

```move
use sui::dynamic_field;
use sui::dynamic_object_field;

// 添加动态字段
dynamic_field::add(&mut parent.id, b"score", 100u64);

// 读取
let score: &u64 = dynamic_field::borrow(&parent.id, b"score");

// 修改
let score: &mut u64 = dynamic_field::borrow_mut(&mut parent.id, b"score");
*score = 200;

// 动态对象字段（子对象，可被直接引用）
dynamic_object_field::add(&mut parent.id, b"child_nft", child_nft);
let nft: &ChildNFT = dynamic_object_field::borrow(&parent.id, b"child_nft");
```

---

## 5. Kiosk（NFT 交易标准）

Kiosk 是 Sui 的 NFT 交易框架，支持强制版税与自定义交易规则：

```move
use sui::kiosk::{Self, Kiosk, KioskOwnerCap};

// 创建 Kiosk（用户的 NFT 展示柜）
let (kiosk, cap) = kiosk::new(ctx);
transfer::share_object(kiosk);
transfer::transfer(cap, tx_context::sender(ctx));

// 将 NFT 放入 Kiosk
kiosk::place(&mut kiosk, &cap, nft);

// 上架（定价出售）
kiosk::list<MyNFT>(&mut kiosk, &cap, nft_id, price);

// 购买（需实现 TransferPolicy，验证是否满足版税规则）
let (nft, request) = kiosk::purchase<MyNFT>(&mut kiosk, nft_id, payment);
transfer_policy::confirm_request(&policy, request);
```

---

## 6. Sui 开发工具链

```bash
# 安装 Sui CLI
cargo install --locked --git https://github.com/MystenLabs/sui.git --branch mainnet sui

# 新建项目
sui move new my_project
cd my_project

# 编译
sui move build

# 测试
sui move test

# 发布到 devnet
sui client publish --gas-budget 100000000

# 切换网络
sui client switch --env mainnet
```

### Move.toml 配置

```toml
[package]
name = "my_project"
version = "0.0.1"
edition = "2024.beta"

[dependencies]
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/mainnet" }

[addresses]
my_project = "0x0"  # 发布前为 0x0，发布后替换为实际地址
```

---

## 参考资源

- [Sui 开发者文档](https://docs.sui.io/)
- [Sui Move 教程](https://docs.sui.io/guides/developer/first-app)
- [Kiosk 文档](https://docs.sui.io/standards/kiosk)
- [Sui dApp Kit](https://sdk.mystenlabs.com/dapp-kit)
