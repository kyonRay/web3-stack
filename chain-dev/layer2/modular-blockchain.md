# 模块化区块链

> **标签：** 模块化、执行层、结算层、共识层、数据可用性

**模块化区块链**将区块链的四个核心职责拆分到专用层，而非由单条链全部承担：

| 层 | 职责 | 以太坊类比 |
|----|------|-----------|
| **执行层** | 执行交易、运行合约、生成状态根 | L2 Rollup |
| **结算层** | 验证证明、解决争议、最终化状态 | 以太坊 L1 |
| **共识层** | 对交易顺序与有效性达成一致 | 以太坊信标链 |
| **数据可用性（DA）层** | 保证交易数据可被获取与验证 | 以太坊 + EIP-4844 / Celestia |

---

## 1. 单体链 vs 模块化

```
单体链（如早期以太坊）：
  同一组节点承担执行 + 共识 + DA
  瓶颈：所有节点都要下载、验证所有数据

模块化：
  执行链（L2） → 只运行 EVM，不参与 L1 共识
  DA 层（Celestia/EIP-4844） → 只保证数据可用，不执行
  结算层（以太坊 L1）→ 只验证证明，不执行批次
```

---

## 2. 以太坊的模块化演进

### The Merge（2022）
- 执行层（op-geth 等）与共识层（Prysm/Lighthouse）分离
- 通过 Engine API 通信

### EIP-4844（Dencun，2024）
- 引入 blob 交易，专用于 L2 DA
- blob 数据约 18 天后可从全节点裁剪（降低存储压力）
- DA 费用与执行 gas 分开计价

### 未来：Danksharding
- Full Danksharding：大量 blob slot（目标 32-64 MB/block）
- DAS（Data Availability Sampling）：节点只下载随机样本，通过纠删码验证全量可用性
- 进一步将以太坊定位为 L2 的结算 + DA 层

---

## 3. 执行层多样化

随着模块化成熟，执行层出现多样化：

| 执行环境 | 代表 | 说明 |
|----------|------|------|
| EVM | Arbitrum、OP Stack | 以太坊兼容 |
| ZK-EVM | zkSync Era、Scroll | ZK 证明的 EVM |
| Cairo VM | StarkNet | ZK 原生 VM |
| SVM（Solana VM） | Eclipse | SVM + 以太坊结算 |
| 自定义 VM | Fuel（FuelVM） | UTXO 并行 VM |

---

## 4. 结算层选择

Rollup 可以选择不同的结算层：

| 结算层 | 代表 Rollup | 安全模型 |
|--------|------------|---------|
| **以太坊 L1** | Arbitrum One、OP Mainnet | 最高安全，共识由以太坊全节点保证 |
| **Arbitrum One（L3）** | Arbitrum Orbit L3 | 继承 Arbitrum One 安全 |
| **Celestia（仅 DA）** | Eclipse | 结算需自建或用其他链 |

---

## 5. 模块化技术栈示例

```
Eclipse（SVM L2）：
  执行：Solana VM（SVM）
  结算：以太坊 L1（ZK 证明验证）
  DA：Celestia

Fuel（高性能 L2）：
  执行：FuelVM（UTXO 并行）
  结算：以太坊 L1（欺诈证明）
  DA：以太坊 blob

AltDA OP Stack：
  执行：EVM（op-geth）
  结算：以太坊 L1
  DA：EigenDA 或 Celestia（非 L1）
```

---

## 参考资源

- [Celestia 文档](https://docs.celestia.org/)
- [Ethereum 路线图（Danksharding）](https://ethereum.org/en/roadmap/danksharding/)
- [Modular Blockchain 入门](https://celestia.org/what-is-modular/)
- [Fuel Network](https://docs.fuel.network/)
