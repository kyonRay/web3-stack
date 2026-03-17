# 链开发专家

本方向面向希望深入参与区块链基础设施开发的工程师，目标是具备独立开发、调试和扩展公链或 Layer2 网络的能力。

## 方向概览

链开发分为两个层次：

- **Layer 1**：构建或定制基础公链——执行客户端、共识层、P2P 网络、以太坊客户端开发、Substrate/Cosmos 框架
- **Layer 2**：在 L1 安全基础上构建扩容方案——Rollup 原理与实现、OP Stack、Arbitrum 生态、ZK Rollup、模块化架构、数据可用性层

## 子目录

| 目录 | 说明 |
|------|------|
| [layer1/](./layer1/) | Layer 1 链开发：以太坊客户端、共识实现、执行层、Substrate/Cosmos |
| [layer2/](./layer2/) | Layer 2 扩容：Rollup、OP Stack、ZK 系、模块化、数据可用性、跨链 |

## 推荐路径

```
Layer 1 基础
└── 以太坊客户端（Geth/Reth）
    ├── 执行层（EVM、状态转换）
    ├── 共识层（Gasper）
    └── P2P 网络

Layer 2 扩容
└── Rollup 原理
    ├── Optimistic: OP Stack / Arbitrum Orbit
    ├── ZK: zkSync / Polygon zkEVM / StarkNet / Scroll
    └── 模块化: DA 层 / Celestia / EigenDA
```

## 技术栈参考

| 工具 | 用途 |
|------|------|
| Go / Rust | 以太坊执行客户端（Geth/Reth）主要语言 |
| Solidity / Yul | EVM 合约与 L2 合约 |
| Rust / C++ | Substrate 框架、高性能节点 |
| Cairo | StarkNet L2 程序 |

## 延伸阅读

- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [L2Beat](https://l2beat.com/) — L2 生态数据与安全假设对比
- [resources/blockchain-fundamentals.md](../resources/blockchain-fundamentals.md)
