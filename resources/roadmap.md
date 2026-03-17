# 三方向学习路线图

本仓库以三个专家方向组织，面向已有区块链基础知识的工程师。

---

## 方向一：链开发专家

### 目标

能够独立开发、调试和扩展以太坊客户端、L2 系统或应用链。

### 推荐学习路径

**Layer 1（2-4 周）**
1. [执行层](../chain-dev/layer1/execution-layer.md) — EVM 执行模型、状态机、操作码
2. [共识层](../chain-dev/layer1/consensus-implementation.md) — PoS Gasper、验证者生命周期
3. [以太坊客户端](../chain-dev/layer1/ethereum-clients.md) — Geth/Reth 架构与开发
4. [P2P 网络](../chain-dev/layer1/p2p-and-networking.md) — devp2p、libp2p、Mempool

**Layer 2（2-4 周）**
5. [Rollup 概览](../chain-dev/layer2/rollup-overview.md) — L2 全景与选型
6. [OP Stack](../chain-dev/layer2/op-stack.md) — 部署自定义 L2
7. [ZK Rollup](../chain-dev/layer2/zk-rollup.md) — ZK 证明系统深入
8. [模块化区块链](../chain-dev/layer2/modular-blockchain.md) + [数据可用性](../chain-dev/layer2/data-availability.md)

**可选深入**
- [Substrate/Cosmos](../chain-dev/layer1/substrate-cosmos.md) — 应用链框架
- [跨链](../chain-dev/layer2/cross-chain.md) — 跨链桥与消息传递

---

## 方向二：合约开发专家

### 目标

能够独立设计、开发、测试和审计主流生态的智能合约。

### EVM 路径（推荐首选，4-8 周）

**语言基础（1-2 周）**
1. [Solidity 语言深入](../smart-contract-dev/evm/solidity/language-deep-dive.md)
2. [开发框架（Foundry）](../smart-contract-dev/evm/frameworks.md)

**安全与标准（2-3 周）**
3. [Token 标准](../smart-contract-dev/evm/token-standards.md)
4. [安全模式与最佳实践](../smart-contract-dev/evm/solidity/best-practices.md)
5. [Gas 优化](../smart-contract-dev/evm/solidity/gas-optimization.md)
6. [测试与审计](../smart-contract-dev/evm/testing-and-auditing.md)

**进阶精通（2-3 周）**
7. [深度安全模式](../smart-contract-dev/evm/solidity/security-patterns.md)
8. [进阶主题（代理/AA/Create2）](../smart-contract-dev/evm/solidity/advanced-topics.md)
9. [EVM 内部原理](../smart-contract-dev/evm/evm-internals.md)

### Solana 路径（2-4 周）

1. [Rust for Solana](../smart-contract-dev/solana/rust-for-solana.md)
2. [Anchor 框架](../smart-contract-dev/solana/anchor-framework.md)
3. [PDA 与 CPI](../smart-contract-dev/solana/pda-and-cpi.md)
4. [SPL Token](../smart-contract-dev/solana/spl-tokens.md)
5. [测试与部署](../smart-contract-dev/solana/testing-and-deployment.md)

### Move 路径（2-3 周）

1. [Move 语言核心](../smart-contract-dev/move/move-language.md)
2. [Sui 开发](../smart-contract-dev/move/sui-development.md) 或 [Aptos 开发](../smart-contract-dev/move/aptos-development.md)
3. [测试与部署](../smart-contract-dev/move/testing-and-deployment.md)

---

## 方向三：DApp 开发专家

### 目标

能够独立构建生产级 DApp 前端，覆盖钱包集成、合约交互、数据索引与部署。

### EVM DApp 路径（2-4 周）

1. [钱包集成（RainbowKit + Wagmi）](../dapp-dev/evm/wallet-integration.md)
2. [合约交互（Viem）](../dapp-dev/evm/contract-interaction.md)
3. [前端框架（Next.js）](../dapp-dev/evm/frontend-frameworks.md)
4. [RPC 节点服务](../dapp-dev/evm/node-services.md)
5. [数据索引（The Graph）](../dapp-dev/evm/data-indexing.md)
6. [部署（Vercel/IPFS）](../dapp-dev/evm/deployment-hosting.md)

### Solana DApp 路径（1-2 周，需有 EVM DApp 基础）

1. [Solana 钱包集成](../dapp-dev/solana/wallet-integration.md)
2. [合约交互（Anchor client）](../dapp-dev/solana/contract-interaction.md)
3. [前端框架](../dapp-dev/solana/frontend-frameworks.md)
4. [数据索引（Helius）](../dapp-dev/solana/data-and-indexing.md)

### Move DApp 路径（1-2 周）

1. [Sui 钱包集成（dApp Kit）](../dapp-dev/move/wallet-integration.md)
2. [合约交互（PTB）](../dapp-dev/move/contract-interaction.md)
3. [前端与索引](../dapp-dev/move/frontend-and-indexing.md)

---

## 推荐资源

### 课程

- [Cyfrin Updraft](https://updraft.cyfrin.io/) — 免费 Solidity 全栈课程
- [Buildspace](https://buildspace.so/) — 项目驱动学习
- [Solana Development Course](https://www.soldev.app/course)

### 实践平台

- [Ethernaut](https://ethernaut.openzeppelin.com/) — Solidity 安全挑战
- [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) — DeFi 攻防
- [Codehawks（竞赛审计）](https://www.codehawks.com/)

### 文档

- [Ethereum 官方文档](https://ethereum.org/en/developers/)
- [Solana 开发文档](https://docs.solana.com/)
- [Sui 文档](https://docs.sui.io/)
- [Aptos 文档](https://aptos.dev/)
