# Web3 技术专家知识库

> 面向已有区块链基础的工程师，系统成为三个方向的 Web3 专家。

---

## 三大专家方向

### 🔗 [链开发专家](./chain-dev/README.md)

从以太坊客户端源码到 Layer 2 系统开发，掌握区块链底层架构。

| 模块 | 主题 |
|------|------|
| **Layer 1** | EVM 执行模型、共识实现（Gasper）、P2P 网络、以太坊客户端（Geth/Reth） |
| **Layer 2** | Optimistic Rollup、ZK Rollup、OP Stack、Arbitrum、模块化区块链、数据可用性 |
| **应用链** | Substrate（Polkadot）、Cosmos SDK、跨链消息（CCIP/LayerZero） |

---

### 📝 [合约开发专家](./smart-contract-dev/README.md)

精通主流生态智能合约开发，尤其 EVM Solidity 深度精通。

| 生态 | 核心技术栈 |
|------|-----------|
| **EVM（重点）** | Solidity 深度精通、Foundry/Hardhat、ERC 标准、安全审计、EVM 原理 |
| **Solana** | Rust + Anchor、PDA/CPI、SPL Token、Token-2022 |
| **Move** | Sui Move（对象模型、PTB）、Aptos Move（资源模型、FA 标准） |

---

### 🖥️ [DApp 开发专家](./dapp-dev/README.md)

构建生产级去中心化应用，覆盖钱包、合约交互、数据索引全链路。

| 生态 | 核心技术栈 |
|------|-----------|
| **EVM** | Wagmi + Viem + RainbowKit、Next.js、The Graph、Alchemy |
| **Solana** | Wallet Adapter、@solana/web3.js、Anchor client、Helius |
| **Move** | Sui dApp Kit、Aptos TS SDK、GraphQL Indexer |

---

## 辅助模块

| 目录 | 内容 |
|------|------|
| [protocols/](./protocols/README.md) | DeFi、NFT、DAO、预言机、稳定币、ZKP、安全审计 |
| [ai-workflows/](./ai-workflows/README.md) | AI 辅助合约审计与代码生成 Prompt |
| [resources/](./resources/README.md) | 区块链基础速查、术语表、学习路线图 |

---

## 快速开始

根据你的目标方向选择入口：

**想做链开发** → [chain-dev/README.md](./chain-dev/README.md)

**想精通合约** → [smart-contract-dev/evm/solidity/README.md](./smart-contract-dev/evm/solidity/README.md)（EVM 首选）

**想做 DApp** → [dapp-dev/evm/README.md](./dapp-dev/evm/README.md)（EVM 首选）

**规划完整路径** → [resources/roadmap.md](./resources/roadmap.md)

---

## 目录结构

```
web3-stack/
├── chain-dev/            # 链开发专家
│   ├── layer1/           #   执行层、共识层、客户端、P2P
│   └── layer2/           #   Rollup、OP Stack、ZK、模块化、DA、跨链
├── smart-contract-dev/   # 合约开发专家
│   ├── evm/              #   Solidity（重点）、框架、标准、审计
│   │   └── solidity/     #     语言深入、最佳实践、安全、高级主题
│   ├── solana/           #   Rust、Anchor、SPL Token、PDA/CPI
│   └── move/             #   Move 语言、Sui、Aptos
├── dapp-dev/             # DApp 开发专家
│   ├── evm/              #   Wagmi、Viem、The Graph、部署
│   ├── solana/           #   Wallet Adapter、Anchor client、Helius
│   └── move/             #   Sui dApp Kit、Aptos SDK
├── protocols/            # 协议与用例速查
├── ai-workflows/         # AI 辅助工作流
└── resources/            # 通用资源
```
