# Arbitrum 生态

> **标签：** arbitrum、nitro、orbit、stylus、bold

Arbitrum 是目前 TVL 最高的以太坊 L2，由 Offchain Labs 开发。生态包括：**Arbitrum One**（旗舰 L2）、**Arbitrum Orbit**（自定义 L2 框架）、**Stylus**（多语言合约运行时）。

---

## 1. Arbitrum Nitro 架构

Nitro 是 Arbitrum 的第二代架构，相比 Classic 大幅提升性能：

```
用户交易
  ↓
Sequencer（AnyTrust 或 Rollup 模式）
  ├── Sequencer Feed（实时广播 L2 交易）
  ├── 执行（geth-based EVM，内嵌 Arbitrum 修改）
  └── 批次压缩（Brotli 压缩，提交到 L1）

L1（以太坊）
  ├── SequencerInbox（接收批次）
  ├── RollupProxy（状态根与欺诈游戏）
  ├── Inbox（L1→L2 消息）
  └── Outbox（L2→L1 消息，提款）
```

### 核心创新

- **Nitro VM**：本质是编译到 WASM 的 Geth，欺诈证明在 L1 上通过 WASM 仿真重放
- **多轮交互式欺诈证明（BOLD）**：二分查找争议区间至单步，L1 只执行 1 条指令
- **Brotli 压缩**：交易数据压缩率高，降低 calldata 成本

---

## 2. 多轮欺诈证明（BOLD）

**BoLD（Bounded Liquidity Delay）** 是 Arbitrum 的最新欺诈证明协议：

1. 挑战者与防御者各提交断言（Assertion）
2. 协议通过二分查找缩小争议至单步
3. L1 合约执行单步 WASM 指令，决定胜负
4. **时间界限**：BoLD 引入确定性提款延迟上限，而非依赖挑战者的及时响应

---

## 3. Arbitrum AnyTrust

AnyTrust 是 Arbitrum Nitro 的变体，专为**数据可用性委员会（DAC）**设计：

- 数据由 DAC 存储和保证，不全量上链（与 Validium 类似）
- **诚实假设**：至少 2 个 DAC 成员诚实
- 优点：大幅降低 DA 成本
- 代表：Arbitrum Nova（面向游戏/社交场景）

---

## 4. Arbitrum Orbit（自定义 L2）

Orbit 允许任何人基于 Arbitrum 技术栈创建 L3 或 L2：

```
以太坊 L1
  └── Arbitrum One (L2)
        ├── L3 Chain A（结算到 Arbitrum One）
        └── L3 Chain B（AnyTrust，游戏链）
```

### 部署 Orbit 链

```bash
# 使用 Nitro Contracts 部署 Orbit 链
git clone https://github.com/OffchainLabs/nitro-contracts
cd nitro-contracts

# 配置
cat > config.ts << 'EOF'
export const config = {
  chainId: 412346,
  chainName: "My Orbit Chain",
  validators: ["0x..."],
  batchPoster: "0x...",
  nativeToken: ethers.constants.AddressZero, // ETH
}
EOF
```

---

## 5. Stylus：多语言合约

Stylus 允许用 **Rust、C、C++** 编写编译到 WASM 的合约，在 Arbitrum 上与 EVM 合约并存：

```rust
// Stylus Rust 合约示例
#[entrypoint]
pub fn counter(input: Vec<u8>) -> ArbResult {
    let count = StorageU256::new(0.into(), 0);
    let mut count = count.load();
    count += U256::from(1);
    StorageU256::new(0.into(), 0).store(count);
    Ok(count.to_be_bytes_vec())
}
```

**优势**：
- WASM 执行约比 EVM 快 10x（计算密集型）
- 可复用 Rust 生态库（密码学、数学库等）
- 与 EVM 合约完全可组合（可相互调用）

---

## 6. 参考资源

- [Arbitrum 文档](https://docs.arbitrum.io/)
- [Nitro 白皮书](https://github.com/OffchainLabs/nitro/blob/master/docs/Nitro-whitepaper.pdf)
- [Stylus 文档](https://docs.arbitrum.io/stylus/stylus-gentle-introduction)
- [BOLD 设计文档](https://github.com/OffchainLabs/bold)
- [Orbit 快速开始](https://docs.arbitrum.io/launch-orbit-chain/orbit-gentle-introduction)
