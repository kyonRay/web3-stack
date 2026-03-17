# EVM 数据索引

> **标签：** the-graph、dune、moralis、链下数据、subgraph

链上历史与事件需要被索引才能支撑复杂的 DApp 界面与分析功能。

---

## 1. The Graph（子图）

The Graph 将链上事件索引为可通过 GraphQL 查询的结构化数据。

### 创建子图

```bash
# 安装 Graph CLI
npm install -g @graphprotocol/graph-cli

# 从合约 ABI 初始化子图
graph init \
  --studio my-subgraph \
  --from-contract 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 \  # USDC
  --network mainnet \
  --abi ./abis/ERC20.json
```

### subgraph.yaml 配置

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: ERC20
    network: mainnet
    source:
      address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
      abi: ERC20
      startBlock: 6082465
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Transfer
        - Account
      abis:
        - name: ERC20
          file: ./abis/ERC20.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
      file: ./src/mapping.ts
```

### schema.graphql

```graphql
type Transfer @entity(immutable: true) {
    id: Bytes!
    from: Account!
    to: Account!
    amount: BigInt!
    blockTimestamp: BigInt!
    transactionHash: Bytes!
}

type Account @entity {
    id: Bytes!
    balance: BigInt!
    transfersFrom: [Transfer!]! @derivedFrom(field: "from")
    transfersTo: [Transfer!]! @derivedFrom(field: "to")
}
```

### mapping.ts（事件处理器）

```typescript
import { Transfer as TransferEvent } from '../generated/ERC20/ERC20'
import { Transfer, Account } from '../generated/schema'

export function handleTransfer(event: TransferEvent): void {
    // 更新或创建账户
    let fromAccount = Account.load(event.params.from)
    if (!fromAccount) {
        fromAccount = new Account(event.params.from)
        fromAccount.balance = BigInt.fromI32(0)
    }
    fromAccount.balance = fromAccount.balance.minus(event.params.value)
    fromAccount.save()

    // 记录转账事件
    let transfer = new Transfer(event.transaction.hash.concatI32(event.logIndex.toI32()))
    transfer.from = event.params.from
    transfer.to = event.params.to
    transfer.amount = event.params.value
    transfer.blockTimestamp = event.block.timestamp
    transfer.transactionHash = event.transaction.hash
    transfer.save()
}
```

### 查询子图

```typescript
const SUBGRAPH_URL = 'https://api.studio.thegraph.com/query/...'

const query = `
    query GetTransfers($user: String!) {
        transfers(
            where: { to: $user }
            orderBy: blockTimestamp
            orderDirection: desc
            first: 10
        ) {
            id
            from { id balance }
            amount
            blockTimestamp
            transactionHash
        }
    }
`

const response = await fetch(SUBGRAPH_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables: { user: userAddress.toLowerCase() } }),
})
const { data } = await response.json()
```

---

## 2. Dune Analytics

Dune 将链上数据同步到数据仓库，用 SQL 做分析：

```sql
-- Dune SQL：查询 USDC 最近 7 天的每日转账量
SELECT
    DATE_TRUNC('day', evt_block_time) as day,
    COUNT(*) as transfer_count,
    SUM(CAST(value AS DOUBLE) / 1e6) as volume_usdc
FROM erc20_ethereum.evt_Transfer
WHERE contract_address = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
    AND evt_block_time >= NOW() - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 1 DESC
```

### 通过 API 读取 Dune 数据

```typescript
// Dune API（需订阅）
const response = await fetch(
    `https://api.dune.com/api/v1/query/${QUERY_ID}/results`,
    { headers: { 'X-Dune-API-Key': DUNE_API_KEY } }
)
const { result } = await response.json()
```

---

## 3. 第三方数据 API

| 服务 | 功能 | 说明 |
|------|------|------|
| **Alchemy** NFT API | NFT 元数据、持仓 | 速度快，免费层慷慨 |
| **Moralis** | NFT/Token/交易历史 | 跨链支持好 |
| **Covalent** | 通用链上数据 API | 200+ 条链 |
| **Unmarshal** | Token 余额、历史 | 多链 |
| **DefiLlama API** | TVL、价格、协议数据 | 免费 DeFi 数据 |

```typescript
// Alchemy NFT API 示例
const response = await fetch(
    `https://eth-mainnet.g.alchemy.com/nft/v3/${ALCHEMY_KEY}/getNFTsForOwner?owner=${address}&withMetadata=true`
)
const { ownedNfts } = await response.json()
```

---

## 4. Viem/Wagmi 读取事件历史

```typescript
// 不依赖索引器，直接读链上历史（适合少量数据）
const logs = await publicClient.getLogs({
    address: CONTRACT_ADDRESS,
    event: parseAbiItem('event Transfer(address indexed from, address indexed to, uint256 value)'),
    fromBlock: BigInt(startBlock),
    toBlock: 'latest',
    args: { from: userAddress },
})
```

**注意**：大范围历史查询性能差，生产环境应使用索引器（The Graph / Alchemy 等）。

---

## 5. 区块浏览器

区块浏览器是最基础的链上数据查询工具，也提供 API 接口：

| 浏览器 | 支持链 | 特色功能 |
|--------|--------|---------|
| **Etherscan** | 以太坊主网及 L2 | 合约验证、ABI 读取、Token 追踪，行业标准 |
| **BlockScout** | 开源，支持 50+ 链 | 可自托管，OP Stack 链默认使用 |
| **OKLink** | 多链（含国内常用链） | 多链聚合，中文界面友好 |
| **NFTScan** | 专注 NFT 数据 | NFT 持仓、交易历史、Collection 分析 |
| **Tenderly** | EVM 链 | 交易模拟、调试、Gas Profile |

### Etherscan API 使用

```typescript
// 获取地址的 ERC-20 转账历史
const res = await fetch(
  `https://api.etherscan.io/api?module=account&action=tokentx` +
  `&address=${address}&startblock=0&endblock=99999999` +
  `&sort=desc&apikey=${ETHERSCAN_KEY}`
)
const { result } = await res.json()

// 获取合约 ABI（用于动态加载）
const abiRes = await fetch(
  `https://api.etherscan.io/api?module=contract&action=getabi` +
  `&address=${CONTRACT_ADDRESS}&apikey=${ETHERSCAN_KEY}`
)
const { result: abi } = await abiRes.json()
```

---

## 参考资源

- [The Graph 文档](https://thegraph.com/docs/)
- [Dune 文档](https://docs.dune.com/)
- [Alchemy NFT API](https://docs.alchemy.com/reference/nft-api-faq)
- [Subgraph Studio](https://thegraph.com/studio/)
- [Etherscan API 文档](https://docs.etherscan.io/)
- [BlockScout 文档](https://docs.blockscout.com/)
- [NFTScan API](https://docs.nftscan.com/)
