# Move DApp 前端与数据索引

> **标签：** sui-indexer、aptos-indexer、graphql、数据查询

---

## 1. Sui 数据索引

### Sui 官方 Indexer（GraphQL API）

```typescript
// Sui GraphQL API
const SUI_GRAPHQL_ENDPOINT = 'https://sui-mainnet.mystenlabs.com/graphql'

const query = `
    query GetNFTsByOwner($owner: SuiAddress!) {
        address(address: $owner) {
            objects {
                nodes {
                    objectId
                    version
                    contents {
                        ... on MoveObject {
                            type { repr }
                            fields
                        }
                    }
                    display {
                        key
                        value
                    }
                }
            }
        }
    }
`

const response = await fetch(SUI_GRAPHQL_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables: { owner: walletAddress } }),
})
const { data } = await response.json()
```

### SuiScan API

```typescript
// SuiScan 提供交易历史、代币持仓等
const response = await fetch(
    `https://suiscan.xyz/api/sui/mainnet/accounts/${address}/tokens`
)
const tokens = await response.json()
```

### 使用 SuiClient 直接查询

```typescript
import { SuiClient } from '@mysten/sui/client'

const client = new SuiClient({ url: 'https://fullnode.mainnet.sui.io:443' })

// 查询持有的特定类型对象
const objects = await client.getOwnedObjects({
    owner: walletAddress,
    filter: {
        StructType: `${PACKAGE_ID}::nft::NFT`,
    },
    options: {
        showContent: true,
        showDisplay: true,
        showType: true,
    },
    cursor: null,
    limit: 20,
})

// 分页处理
while (objects.hasNextPage) {
    const nextPage = await client.getOwnedObjects({
        owner: walletAddress,
        cursor: objects.nextCursor,
    })
    // 处理 nextPage
}

// 订阅实时事件（WebSocket）
const unsubscribe = await client.subscribeEvent({
    filter: { Package: PACKAGE_ID },
    onMessage: (event) => {
        console.log('事件类型:', event.type)
        console.log('事件数据:', event.parsedJson)
    },
})
```

---

## 2. Aptos 数据索引

### Aptos Indexer（GraphQL）

```typescript
const APTOS_INDEXER_ENDPOINT = 'https://api.mainnet.aptoslabs.com/v1/graphql'

// 查询账户持有的 NFT（数字资产）
const query = `
    query GetNFTs($owner: String!) {
        current_token_ownerships_v2(
            where: {
                owner_address: { _eq: $owner }
                amount: { _gt: "0" }
            }
            limit: 50
        ) {
            current_token_data {
                token_name
                token_uri
                collection_id
                current_collection {
                    collection_name
                    creator_address
                }
            }
            amount
        }
    }
`

const response = await fetch(APTOS_INDEXER_ENDPOINT, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables: { owner: walletAddress } }),
})
const { data } = await response.json()
```

### 查询代币余额

```typescript
// Aptos Indexer：查询所有 Fungible Asset 余额
const faQuery = `
    query GetFungibleAssets($owner: String!) {
        current_fungible_asset_balances(
            where: { owner_address: { _eq: $owner }, amount: { _gt: "0" } }
        ) {
            amount
            asset_type
            metadata {
                name
                symbol
                decimals
            }
        }
    }
`
```

### Aptos REST API

```typescript
const aptosClient = new Aptos(new AptosConfig({ network: Network.MAINNET }))

// 获取账户资源
const resources = await aptosClient.getAccountResources({ accountAddress })
const coinStore = resources.find(r => r.type.includes('CoinStore<AptosCoin>'))
console.log('APT 余额:', coinStore?.data.coin.value)

// 获取交易历史
const txs = await aptosClient.getAccountTransactions({
    accountAddress,
    options: { limit: 25 },
})
```

---

## 3. 完整 Sui DApp 示例

```tsx
// app/nft-gallery/page.tsx
'use client'
import { useCurrentAccount, useSuiClientQuery } from '@mysten/dapp-kit'

export default function NFTGallery() {
    const account = useCurrentAccount()

    const { data: nfts, isLoading } = useSuiClientQuery(
        'getOwnedObjects',
        {
            owner: account?.address ?? '',
            filter: { StructType: `${NFT_PACKAGE}::nft::NFT` },
            options: { showDisplay: true, showContent: true },
        },
        { enabled: !!account }
    )

    if (isLoading) return <div>加载中...</div>

    return (
        <div className="grid grid-cols-3 gap-4">
            {nfts?.data.map((nft) => {
                const display = nft.data?.display?.data
                return (
                    <div key={nft.data?.objectId} className="rounded-lg overflow-hidden">
                        <img src={display?.image_url} alt={display?.name} />
                        <p className="p-2 font-bold">{display?.name}</p>
                    </div>
                )
            })}
        </div>
    )
}
```

---

## 参考资源

- [Sui GraphQL API](https://docs.sui.io/references/sui-api/sui-graphql)
- [Aptos Indexer GraphQL](https://aptos.dev/indexer/api/graphql-queries)
- [SuiScan API](https://suiscan.xyz/)
- [Sui dApp Kit：useSuiClientQuery](https://sdk.mystenlabs.com/dapp-kit/rpc-hooks)
