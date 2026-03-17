# EVM DApp 开发

面向以太坊及所有 EVM 兼容链（Arbitrum、Optimism、Base、Polygon、zkSync 等）的前端与全栈 DApp 开发。

## 文档列表

| 文档 | 说明 |
|------|------|
| [dapp-architecture.md](./dapp-architecture.md) | DApp 架构：前端↔链端模型、RPC 容错、代理合约、去中心化存储 |
| [wallet-integration.md](./wallet-integration.md) | 钱包集成：MetaMask/imToken/Safe/Ledger/AA 钱包分类全览 |
| [contract-interaction.md](./contract-interaction.md) | Ethers.js、Web3.js、Viem、ABI 编码与事件监听 |
| [frontend-frameworks.md](./frontend-frameworks.md) | React + Wagmi、Next.js、Vue、TailwindCSS |
| [data-indexing.md](./data-indexing.md) | The Graph、Dune、Etherscan/BlockScout/NFTScan 区块浏览器 |
| [node-services.md](./node-services.md) | Infura、Alchemy、QuickNode、公共 RPC |
| [deployment-hosting.md](./deployment-hosting.md) | Vercel、IPFS/Arweave/EthStorage 去中心化托管 |

## 推荐技术栈

### 现代标配（2024）

```
React / Next.js
  + Wagmi（钱包 Hooks）
  + Viem（底层 EVM 交互）
  + RainbowKit（钱包 UI）
  + TailwindCSS（样式）
  + The Graph（数据索引）
  + Alchemy / Infura（RPC）
```

### 多链支持

```typescript
// Wagmi 多链配置示例
import { createConfig, http } from 'wagmi'
import { mainnet, arbitrum, optimism, base } from 'wagmi/chains'

const config = createConfig({
  chains: [mainnet, arbitrum, optimism, base],
  transports: {
    [mainnet.id]: http('https://mainnet.infura.io/v3/YOUR_KEY'),
    [arbitrum.id]: http('https://arb-mainnet.g.alchemy.com/v2/YOUR_KEY'),
    [optimism.id]: http(),
    [base.id]: http(),
  },
})
```

## 推荐学习顺序

1. [wallet-integration.md](./wallet-integration.md) — 先打通「连接钱包」
2. [contract-interaction.md](./contract-interaction.md) — 实现读写合约
3. [frontend-frameworks.md](./frontend-frameworks.md) — 完整 React DApp
4. [node-services.md](./node-services.md) — 生产级 RPC 配置
5. [data-indexing.md](./data-indexing.md) — 历史数据与分析
6. [deployment-hosting.md](./deployment-hosting.md) — 上线部署
