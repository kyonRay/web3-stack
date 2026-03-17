# Rollup 原理与 L2 全景

> **标签：** rollup、optimistic、zk、状态通道、plasma、validium

本文是 Layer 2 方向的入口文档，建立 L2 扩容的整体认知框架。

---

## 1. 为什么需要 L2

以太坊 L1 有意限制吞吐（约 15–30 TPS），以保持去中心化与安全。区块链不可能三角：

- **去中心化**：L1 的核心目标
- **安全**：L1 的核心目标
- **可扩展性**：L2 负责解决

L2 方案：在链下执行大量交易，将执行结果（状态根 + 证明/数据）提交到 L1，借助 L1 的安全性解决争议。

---

## 2. Optimistic Rollup

### 2.1 原理

1. Sequencer 在链下批量执行交易，生成新状态根
2. 将压缩后的交易数据 + 状态根提交到 L1 合约
3. **乐观假设**：状态根有效，除非有人提交欺诈证明
4. **挑战窗口**（7 天）：任何人可重放争议交易，证明状态根错误
5. 若欺诈成立 → 回滚 + 惩罚 Sequencer

### 2.2 欺诈证明类型

| 类型 | 代表 | 说明 |
|------|------|------|
| **多轮交互式** | Arbitrum Nitro | 二分查找定位单条指令，L1 只执行一条指令 |
| **单轮** | Optimism Cannon | L1 重放整个争议区块，更简单但更贵 |

### 2.3 主要实现

- **Arbitrum One** — 多轮欺诈证明，L2 TVL 领先，[arbitrum-ecosystem.md](./arbitrum-ecosystem.md)
- **OP Mainnet** — 单轮证明，OP Stack 生态，[op-stack.md](./op-stack.md)
- **Base** — Coinbase 运营，OP Stack

### 2.4 关键参数

| 参数 | 典型值 |
|------|--------|
| 提款延迟 | 7 天（挑战期） |
| EVM 兼容性 | 完整字节码兼容 |
| 数据可用性 | L1 calldata / blob（EIP-4844） |
| 信任假设 | 至少 1 个诚实验证者 |

---

## 3. ZK Rollup

### 3.1 原理

1. Prover 在链下执行交易并生成**有效性证明**（SNARK/STARK）
2. 证明以密码学方式保证新状态根是批次执行的正确结果
3. L1 验证合约只需验证证明（而非重放交易），**无需挑战窗口**

### 3.2 ZK-EVM 兼容性谱系（Vitalik 分类）

| Type | 说明 | 代表 |
|------|------|------|
| **Type 1** | 完全以太坊等价（相同哈希函数） | Taiko |
| **Type 2** | EVM 等价（不同状态表示） | Scroll |
| **Type 2.5** | EVM 兼容（Gas 成本不同） | |
| **Type 3** | 大部分兼容（少量差异） | Polygon zkEVM |
| **Type 4** | 编译到 ZK 友好 VM | zkSync Era、StarkNet |

### 3.3 主要实现

- **zkSync Era** — Type 4，[zk-rollup.md](./zk-rollup.md)
- **Polygon zkEVM** — Type 3，[zk-rollup.md](./zk-rollup.md)
- **StarkNet** — Cairo VM，[zk-rollup.md](./zk-rollup.md)
- **Scroll** — Type 2，[zk-rollup.md](./zk-rollup.md)

---

## 4. 状态通道

两方在 L1 锁定资金，在链下交换签名状态，只在最终结算时上链：

- **优点**：即时确认、极低费用
- **缺点**：需双方在线、不适合开放参与
- **适合**：微支付、游戏步骤
- **代表**：Lightning Network（BTC）、Raiden（ERC-20）

---

## 5. Plasma 与 Validium

**Plasma**：子链向 L1 提交状态承诺，通过 Merkle 证明退出。已基本被 Rollup 取代。

**Validium**：ZK 证明上链，但数据**不**上链（链下 DA）：
- 比 ZK Rollup 更便宜（省去 blob 费用）
- 安全性依赖数据可用性委员会（DAC）
- **Volition**（StarkEx）允许用户每笔交易选择 rollup 或 validium 模式

---

## 6. EIP-4844 对 L2 的影响

Proto-Danksharding（2024 年 3 月 Dencun 升级）：
- 新增 blob 交易类型，专用于 L2 数据发布
- blob 数据约 18 天后可从全节点删除（但在此期间可下载验证）
- 独立 blob 费用市场，大幅降低 L2 提交数据成本（10–100x 降价）

---

## 7. Optimistic vs ZK 选型建议

| 场景 | 推荐 |
|------|------|
| 快速上线、完整 EVM 兼容 | Optimistic（OP Stack 或 Arbitrum Orbit） |
| 需要快速提款、高频交易 | ZK Rollup |
| 隐私 DeFi | ZK（StarkEx/Validium） |
| 自定义 VM 或语言 | StarkNet（Cairo）/ zkSync（Boojum） |

## 参考资源

- [Vitalik：不完整的 Rollup 指南](https://vitalik.eth.limo/general/2021/01/05/rollup.html)
- [L2Beat](https://l2beat.com/)
- [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)
