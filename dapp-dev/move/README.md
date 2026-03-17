# Move DApp 开发（Sui / Aptos）

Sui 和 Aptos 各自提供了完整的前端 SDK，Sui 有官方的 **Sui dApp Kit**（React Hooks），Aptos 有 **Aptos SDK** 与 **Aptos Wallet Adapter**。

## 文档列表

| 文档 | 说明 |
|------|------|
| [wallet-integration.md](./wallet-integration.md) | Sui Wallet Kit、Surf、Aptos Wallet Adapter |
| [contract-interaction.md](./contract-interaction.md) | Sui SDK、Sui dApp Kit、PTB、Aptos SDK |
| [frontend-and-indexing.md](./frontend-and-indexing.md) | Sui Indexer、Aptos Indexer、GraphQL API |

## Sui 推荐技术栈

```
React / Next.js
  + @mysten/dapp-kit（Sui 官方 React Hooks）
  + @mysten/sui（Sui SDK）
  + @mysten/wallet-standard（钱包标准）
  + Sui Indexer / Suiscan API（数据）
```

## Aptos 推荐技术栈

```
React / Next.js
  + @aptos-labs/wallet-adapter-react（钱包 Hooks）
  + @aptos-labs/ts-sdk（Aptos SDK v2）
  + Aptos Indexer GraphQL（数据）
```

## 推荐学习顺序

1. [wallet-integration.md](./wallet-integration.md) — 接入钱包
2. [contract-interaction.md](./contract-interaction.md) — 调用 Move 合约，理解 PTB（Sui）
3. [frontend-and-indexing.md](./frontend-and-indexing.md) — 数据展示与索引

## Sui PTB（可编程交易块）

PTB 是 Sui 的独特能力：一个交易可以链式调用多个合约函数，前一步的输出直接作为后一步的输入，无需多笔交易，实现原子性的复杂操作。

```typescript
// PTB 示例：分割 Coin → 转账 → 调用合约
const tx = new Transaction()
const [coin] = tx.splitCoins(tx.gas, [1000n])
tx.transferObjects([coin], recipientAddress)
tx.moveCall({
  target: '0xPACKAGE::module::function',
  arguments: [tx.object(objectId)],
})
await client.signAndExecuteTransaction({ signer: keypair, transaction: tx })
```

## 参考资源

- [Sui dApp Kit 文档](https://sdk.mystenlabs.com/dapp-kit)
- [Sui TypeScript SDK](https://sdk.mystenlabs.com/typescript)
- [Aptos TypeScript SDK](https://aptos.dev/sdks/ts-sdk/)
- [Aptos Wallet Adapter](https://github.com/aptos-labs/aptos-wallet-adapter)
