# DApp 开发专家

本方向面向希望构建去中心化应用的前端/全栈工程师，覆盖钱包集成、合约交互、前端框架、数据索引、节点服务与部署，分 EVM、Solana、Move 三个生态。

## 方向概览

| 子目录 | 生态 | 核心工具栈 | 说明 |
|--------|------|-----------|------|
| [evm/](./evm/) | EVM 生态 | Ethers.js / Viem / Wagmi / RainbowKit | 以太坊及所有 EVM 兼容链 DApp |
| [solana/](./solana/) | Solana | @solana/web3.js / Anchor client / Wallet Adapter | Solana DApp |
| [move/](./move/) | Sui/Aptos | Sui SDK / Sui dApp Kit / Aptos SDK | Move 生态 DApp |

## DApp 开发通用能力

无论哪个生态，DApp 开发都需要以下核心能力：

1. **钱包集成** — 连接钱包、获取账户、签名交易
2. **合约交互** — 读取链上状态（view/query）、发送交易（write）、监听事件
3. **前端工程** — 状态管理、异步处理（pending/error/success）、多链切换
4. **数据索引** — 链上历史数据、实时数据流
5. **节点与 RPC** — 选择节点服务、处理限流与故障转移
6. **部署与托管** — 前端部署、去中心化托管选项

## 选择生态

```
DApp 开发
├── EVM（推荐首选）
│   ├── 生态最大，库与文档最丰富
│   ├── Wagmi + Viem + RainbowKit 是现代标配
│   └── 支持所有 EVM 兼容链
├── Solana
│   ├── 高性能用户体验（快速确认、低费用）
│   ├── Wallet Adapter 统一多种 Solana 钱包
│   └── Anchor client 简化合约调用
└── Move（Sui / Aptos）
    ├── PTB（可编程交易块）提供更强的原子组合能力
    ├── Sui dApp Kit 提供 React Hooks
    └── 生态相对新，需关注文档更新
```

## 延伸阅读

- [smart-contract-dev/](../smart-contract-dev/) — 理解你调用的合约
- [protocols/](../protocols/) — DeFi、NFT、DAO 等应用场景
