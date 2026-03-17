# Solana DApp 开发

Solana DApp 开发以 **@solana/web3.js**（或新版 `@solana/kit`）为核心，配合 **Wallet Adapter** 统一多钱包支持，Anchor 项目可使用生成的客户端直接调用合约。

## 文档列表

| 文档 | 说明 |
|------|------|
| [wallet-integration.md](./wallet-integration.md) | Phantom、Backpack、Wallet Adapter 统一接入 |
| [contract-interaction.md](./contract-interaction.md) | @solana/web3.js、Anchor client、交易构建 |
| [frontend-frameworks.md](./frontend-frameworks.md) | React + @solana/wallet-adapter-react、Solana dApp scaffold |
| [data-and-indexing.md](./data-and-indexing.md) | Helius、Solana FM、DAS API、RPC 订阅 |

## 推荐技术栈

```
React / Next.js
  + @solana/wallet-adapter-react（钱包 Hooks）
  + @solana/wallet-adapter-wallets（多钱包支持）
  + @solana/web3.js 或 @solana/kit（链交互）
  + Anchor client（合约调用，如使用 Anchor 开发）
  + Helius / QuickNode（RPC + 增强 API）
```

## 推荐学习顺序

1. [wallet-integration.md](./wallet-integration.md) — Wallet Adapter 是 Solana DApp 的标配起点
2. [contract-interaction.md](./contract-interaction.md) — 理解 Solana 交易结构与 Anchor client
3. [frontend-frameworks.md](./frontend-frameworks.md) — 完整 React DApp
4. [data-and-indexing.md](./data-and-indexing.md) — DAS API、Compressed NFT 等数据索引

## Solana 特有注意事项

- **交易结构**：Solana 交易由多条指令组成，每条指令指定程序、账户列表与数据。
- **账户预创建**：部分账户（如 ATA）需要提前创建，前端需处理「创建账户 → 调用合约」的复合交易。
- **计算单元**：复杂交易需要手动设置 `ComputeBudget`，否则可能超出默认限制失败。
- **优先费**：高峰期需添加 priority fee（`SetComputeUnitPrice`）确保交易入块。

## 参考资源

- [Solana Web3.js 文档](https://solana-labs.github.io/solana-web3.js/)
- [Wallet Adapter 文档](https://github.com/solana-labs/wallet-adapter)
- [Helius 文档](https://docs.helius.dev/)
- [Solana Cookbook](https://solanacookbook.com/)
