# Solana 数据索引

> **标签：** helius、solana-fm、das-api、compressed-nft

Solana 的高吞吐（50,000 TPS）使得传统轮询效率低下，专业索引服务是必需的。

---

## 1. Helius（推荐）

Helius 是最全面的 Solana RPC + 索引 API 提供商：

### Digital Asset Standard（DAS）API

DAS 是 Solana 的 NFT/资产统一索引标准，用于查询 NFT 持仓：

```typescript
// 获取钱包持有的所有 NFT（包括 Compressed NFT）
const response = await fetch(
    `https://mainnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}`,
    {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            jsonrpc: '2.0',
            id: 'my-id',
            method: 'getAssetsByOwner',
            params: {
                ownerAddress: walletAddress,
                page: 1,
                limit: 50,
                displayOptions: { showFungible: false },
            },
        }),
    }
)
const { result } = await response.json()
console.log('NFT 总数:', result.total)
```

### Helius Enhanced Transactions

获取人类可读的交易历史：

```typescript
const response = await fetch(
    `https://api.helius.xyz/v0/addresses/${address}/transactions?api-key=${HELIUS_API_KEY}&type=NFT_SALE`
)
const transactions = await response.json()
// 返回解析好的交易（如 NFT Sale 含买家、卖家、价格）
```

### Helius Webhooks

实时监听链上事件（替代 WebSocket 轮询）：

```typescript
// 创建 Webhook（监控特定地址或程序）
const response = await fetch('https://api.helius.xyz/v0/webhooks?api-key=YOUR_KEY', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        webhookURL: 'https://your-server.com/webhook',
        transactionTypes: ['NFT_SALE', 'TOKEN_TRANSFER'],
        accountAddresses: [PROGRAM_ID, USER_WALLET],
        webhookType: 'enhanced',
    }),
})
```

---

## 2. Solana FM

Solana FM 是链上数据浏览器与 API：

```typescript
// 查询账户交易历史
const response = await fetch(
    `https://hyper.solana.fm/v2/signatures?account=${address}&inflow=true`,
    { headers: { 'Content-Type': 'application/json' } }
)
const { result } = await response.json()
```

---

## 3. 直接 RPC 查询（基础场景）

```typescript
import { Connection, PublicKey } from '@solana/web3.js'

const connection = new Connection('https://api.mainnet-beta.solana.com')

// 获取历史交易签名列表
const signatures = await connection.getSignaturesForAddress(
    new PublicKey(address),
    { limit: 10 }
)

// 获取交易详情
const txs = await connection.getTransactions(
    signatures.map(s => s.signature),
    { commitment: 'confirmed', maxSupportedTransactionVersion: 0 }
)
```

---

## 4. Token 余额查询

```typescript
import { getAccount, getAssociatedTokenAddress } from '@solana/spl-token'

// 查询特定代币余额
async function getTokenBalance(
    connection: Connection,
    mint: PublicKey,
    owner: PublicKey
): Promise<bigint> {
    try {
        const ata = await getAssociatedTokenAddress(mint, owner)
        const account = await getAccount(connection, ata)
        return account.amount
    } catch {
        return 0n // ATA 不存在（余额为 0）
    }
}

// 查询所有 SPL Token 余额（使用 getParsedTokenAccountsByOwner）
const tokenAccounts = await connection.getParsedTokenAccountsByOwner(
    new PublicKey(owner),
    { programId: TOKEN_PROGRAM_ID }
)

tokenAccounts.value.forEach(({ account }) => {
    const info = account.data.parsed.info
    console.log('Mint:', info.mint)
    console.log('Balance:', info.tokenAmount.uiAmountString)
})
```

---

## 5. 服务对比

| 服务 | 免费层 | 特色 |
|------|--------|------|
| **Helius** | 1M 请求/月 | DAS API、Webhooks、增强交易 |
| **QuickNode** | 1M 请求/月 | 低延迟、Streams（WebSocket） |
| **Triton** | — | 高性能，机构级 |
| **Solana FM** | 免费 | 交易历史浏览器 |
| **公共 RPC** | 免费 | 有速率限制，开发用 |

---

## 参考资源

- [Helius 文档](https://docs.helius.dev/)
- [DAS API 规范](https://developers.metaplex.com/bubblegum/digital-asset-standard-api)
- [Solana 公共 RPC](https://solana.com/rpc)
