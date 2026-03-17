# Layer 2 扩容开发

Layer 2 在 L1 安全基础上，将执行移至链下以实现高吞吐与低成本，同时通过欺诈证明或有效性证明继承 L1 的安全性。

## 文档列表

| 文档 | 说明 |
|------|------|
| [rollup-overview.md](./rollup-overview.md) | Rollup 原理：乐观 vs ZK、状态通道、Plasma、Validium |
| [op-stack.md](./op-stack.md) | OP Stack 架构：Optimism / Base / 自定义 L2 |
| [arbitrum-ecosystem.md](./arbitrum-ecosystem.md) | Arbitrum Nitro、Orbit、Stylus |
| [zk-rollup.md](./zk-rollup.md) | ZK Rollup 系：zkSync Era、Polygon zkEVM、StarkNet、Scroll |
| [modular-blockchain.md](./modular-blockchain.md) | 模块化区块链：执行/结算/共识/DA 分层 |
| [data-availability.md](./data-availability.md) | 数据可用性：EIP-4844、Celestia、EigenDA、Avail |
| [cross-chain.md](./cross-chain.md) | 跨链桥与消息传递：安全假设、主流协议 |

## 建议学习顺序

1. [rollup-overview.md](./rollup-overview.md) — 建立 L2 全景
2. [op-stack.md](./op-stack.md) 或 [arbitrum-ecosystem.md](./arbitrum-ecosystem.md) — 从乐观 Rollup 入手
3. [zk-rollup.md](./zk-rollup.md) — ZK 系统深入
4. [modular-blockchain.md](./modular-blockchain.md) + [data-availability.md](./data-availability.md) — 模块化架构
5. [cross-chain.md](./cross-chain.md) — 跨链与互操作

## L2 生态对比

| | Optimistic Rollup | ZK Rollup |
|--|-------------------|-----------|
| **代表** | Arbitrum、Optimism、Base | zkSync Era、Polygon zkEVM、StarkNet |
| **提款延迟** | 7 天（挑战期） | 分钟级 |
| **EVM 兼容** | 完整兼容 | 部分至完整 |
| **证明方式** | 欺诈证明（可选） | 有效性证明（必选） |
| **Prover 成本** | 低 | 高 |

## 参考资源

- [L2Beat](https://l2beat.com/) — 实时 TVL、风险评分、DA 方案对比
- [Ethereum L2 文档](https://ethereum.org/en/layer-2/)
- [Rollup Centric Roadmap — Vitalik](https://ethereum-magicians.org/t/a-rollup-centric-ethereum-roadmap/4698)
