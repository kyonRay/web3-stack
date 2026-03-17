# Substrate 与 Cosmos SDK

> **标签：** substrate、cosmos、polkadot、tendermint、链框架

除以太坊生态外，**Substrate**（Polkadot 生态）和 **Cosmos SDK**（Cosmos 生态）是构建应用链（Application-Specific Blockchain）的主流框架。

---

## 1. Substrate（Polkadot 生态）

### 1.1 概述

- **语言**：Rust
- **框架**：由 Parity Technologies 开发，Polkadot 与 Kusama 均基于 Substrate 构建
- **核心理念**：通过 FRAME（Framework for Runtime Aggregation of Modularized Entities）将链逻辑模块化为 **Pallets**

### 1.2 核心架构

```
Substrate 节点
├── 网络层（libp2p）
├── 共识（可插拔：GRANDPA / BABE / Aura / PoA）
├── Runtime（核心业务逻辑，编译为 WASM）
│   ├── Pallet Balances（余额管理）
│   ├── Pallet Staking（质押）
│   ├── Pallet Contracts（EVM 兼容或 ink! 合约）
│   └── 自定义 Pallets
└── 客户端（RPC、数据库）
```

Runtime 以 WASM 形式存储在链上，可通过治理**无分叉升级**。

### 1.3 开发 Pallet

```rust
#[pallet::pallet]
pub struct Pallet<T>(_);

#[pallet::storage]
pub type Counter<T: Config> = StorageValue<_, u32, ValueQuery>;

#[pallet::call]
impl<T: Config> Pallet<T> {
    #[pallet::weight(10_000)]
    pub fn increment(origin: OriginFor<T>) -> DispatchResult {
        let _who = ensure_signed(origin)?;
        Counter::<T>::mutate(|c| *c += 1);
        Ok(())
    }
}
```

### 1.4 与 Polkadot 的关系

- **中继链（Relay Chain）**：Polkadot 主链，提供共享安全
- **平行链（Parachain）**：基于 Substrate 的应用链，租用中继链的安全性
- **XCMP（跨链消息传递）**：平行链之间通信

### 1.5 参考资源

- [Substrate 文档](https://docs.substrate.io/)
- [FRAME Pallets](https://docs.substrate.io/reference/frame-pallets/)
- [Polkadot SDK](https://github.com/paritytech/polkadot-sdk)

---

## 2. Cosmos SDK

### 2.1 概述

- **语言**：Go（CosmWasm 合约用 Rust）
- **共识**：CometBFT（原 Tendermint BFT）——即时终局性
- **核心理念**：每条链完全主权，通过 **IBC 协议**互操作

### 2.2 核心架构

```
Cosmos 节点
├── CometBFT（共识 + P2P）
│   └── ABCI（Application BlockChain Interface）
└── Cosmos SDK 应用
    ├── Baseapp（消息路由）
    ├── 模块（x/bank、x/staking、x/gov...）
    └── Store（IAVL 树状态存储）
```

### 2.3 CometBFT 共识

- 基于 PBFT，验证者轮流出块（Round-Robin）
- **即时终局性**：区块一旦提交就不可回滚
- 需要 2/3 超多数投票才能提交区块

### 2.4 开发 Cosmos 模块

```go
// 定义 Keeper（存储访问层）
type Keeper struct {
    storeKey storetypes.StoreKey
    cdc      codec.BinaryCodec
}

// 消息处理
func (k Keeper) MsgSetValue(
    ctx sdk.Context,
    msg *types.MsgSetValue,
) (*types.MsgSetValueResponse, error) {
    store := ctx.KVStore(k.storeKey)
    store.Set([]byte(msg.Key), []byte(msg.Value))
    return &types.MsgSetValueResponse{}, nil
}
```

### 2.5 IBC（Inter-Blockchain Communication）

IBC 允许不同 Cosmos 链之间传递消息与代币：

- **Light Client**：验证对方链的区块头
- **Channel + Port**：类似网络套接字，建立双向通道
- **Packet**：携带数据的消息单元，需接收方确认

### 2.6 CosmWasm

- 基于 WASM 的智能合约运行时，合约用 **Rust** 编写
- 可在任何支持 CosmWasm 的 Cosmos 链上部署

### 2.7 参考资源

- [Cosmos SDK 文档](https://docs.cosmos.network/)
- [CometBFT 文档](https://docs.cometbft.com/)
- [IBC 协议规范](https://github.com/cosmos/ibc)
- [CosmWasm 文档](https://docs.cosmwasm.com/)

---

## 3. Substrate vs Cosmos SDK 对比

| 维度 | Substrate | Cosmos SDK |
|------|-----------|-----------|
| **语言** | Rust | Go（合约用 Rust） |
| **共识** | 可插拔（GRANDPA/BABE 等） | CometBFT（BFT，即时终局） |
| **互操作** | XCMP（Polkadot 内）/ XCM | IBC（任意 Cosmos 链） |
| **升级** | 无分叉 WASM Runtime 升级 | 需要节点升级 |
| **安全模型** | 可租用中继链安全 | 各链独立验证者集 |
| **适用场景** | DeFi Hub、NFT 链、Polkadot 平行链 | 应用链、交易所链、DeFi 链 |
