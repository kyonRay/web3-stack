# RPC 节点服务

> **标签：** rpc、infura、alchemy、quicknode、节点

DApp 和后端通过 **RPC（JSON-RPC）** 与链通信。可自建全节点，或使用 Infura、Alchemy、QuickNode 等托管 RPC。

---

## 1. JSON-RPC 标准

以太坊 JSON-RPC 是与节点通信的标准接口：

```typescript
// 原始 JSON-RPC 调用（了解原理）
const response = await fetch(RPC_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        jsonrpc: '2.0',
        id: 1,
        method: 'eth_blockNumber',
        params: [],
    }),
})
const { result } = await response.json()
console.log('最新区块:', parseInt(result, 16))
```

### 常用 RPC 方法

| 方法 | 说明 |
|------|------|
| `eth_blockNumber` | 最新区块高度 |
| `eth_getBalance` | ETH 余额 |
| `eth_call` | 调用 view 函数（不上链） |
| `eth_sendRawTransaction` | 广播已签名交易 |
| `eth_getTransactionReceipt` | 交易收据（含 gas 消耗、logs）|
| `eth_getLogs` | 按条件筛选历史日志 |
| `eth_getStorageAt` | 读取合约 storage slot |
| `debug_traceTransaction` | 完整 EVM 执行追踪（归档节点）|
| `eth_subscribe` | WebSocket 实时订阅（新区块/日志）|

---

## 2. 主流托管 RPC 服务

### Alchemy

```typescript
import { createAlchemyWeb3 } from '@alch/alchemy-web3'

const alchemyUrl = `https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`

// Viem 配置
const publicClient = createPublicClient({
    chain: mainnet,
    transport: http(alchemyUrl),
})

// Alchemy 增强 API（额外功能）
const response = await fetch(
    `${alchemyUrl}`,
    {
        method: 'POST',
        body: JSON.stringify({
            method: 'alchemy_getTokenBalances',
            params: [address, 'erc20'],
        }),
    }
)
```

**Alchemy 特有功能**：
- `alchemy_getTokenBalances`：批量查 ERC-20 余额
- `alchemy_getAssetTransfers`：资产转移历史
- NFT API、Webhook、Gas Manager
- Transact API（私有 mempool）

### Infura

```typescript
const infuraUrl = `https://mainnet.infura.io/v3/${INFURA_PROJECT_ID}`

// 支持 WebSocket
const wsUrl = `wss://mainnet.infura.io/ws/v3/${INFURA_PROJECT_ID}`
```

### QuickNode

```typescript
const quicknodeUrl = `https://xxx.quiknode.pro/${QUICKNODE_KEY}/`

// QuickNode 提供 Solana/Aptos 等非 EVM 链
```

### 公共 RPC（测试/开发用）

```typescript
// 免费，有速率限制，不适合生产
const publicRPCs = {
    mainnet: 'https://cloudflare-eth.com',
    arbitrum: 'https://arb1.arbitrum.io/rpc',
    optimism: 'https://mainnet.optimism.io',
    base: 'https://mainnet.base.org',
}
```

---

## 3. 多 RPC 故障转移

生产环境需配置多个 RPC 端点，防止单点故障：

```typescript
import { createPublicClient, fallback, http } from 'viem'

const publicClient = createPublicClient({
    chain: mainnet,
    transport: fallback([
        http(`https://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`),
        http(`https://mainnet.infura.io/v3/${INFURA_KEY}`),
        http('https://cloudflare-eth.com'), // 最后兜底
    ]),
})
```

---

## 4. WebSocket 实时订阅

```typescript
import { createPublicClient, webSocket } from 'viem'

const wsClient = createPublicClient({
    chain: mainnet,
    transport: webSocket(`wss://eth-mainnet.g.alchemy.com/v2/${ALCHEMY_KEY}`),
})

// 监听新区块
const unwatch = wsClient.watchBlocks({
    onBlock: (block) => {
        console.log('新区块:', block.number)
    },
})

// 监听合约事件（实时）
const unwatch2 = wsClient.watchContractEvent({
    address: UNISWAP_V3_POOL,
    abi: poolAbi,
    eventName: 'Swap',
    onLogs: (logs) => {
        logs.forEach(log => console.log('Swap:', log.args))
    },
})
```

---

## 5. 速率限制处理

```typescript
import { createPublicClient, http } from 'viem'

const publicClient = createPublicClient({
    chain: mainnet,
    transport: http(RPC_URL, {
        retryCount: 3,          // 自动重试次数
        retryDelay: 1000,       // 重试间隔（ms）
        timeout: 10_000,        // 超时（ms）
        batch: {
            multicall: true,    // 启用 multicall 批量
        },
    }),
})
```

---

## 6. 服务对比

| 服务 | 免费层 | 特色功能 | 最适合场景 |
|------|--------|---------|-----------|
| **Alchemy** | 300M CU/月 | 增强 API、NFT、Webhook | DApp 全栈 |
| **Infura** | 100K req/天 | 稳定、企业合规 | 生产环境 |
| **QuickNode** | 1M req/月 | 多链、低延迟 | 高性能 |
| **Tenderly** | — | 调试、模拟、监控 | 开发调试 |
| **公共 RPC** | 免费无限 | — | 开发/测试 |

---

## 参考资源

- [Ethereum JSON-RPC 规范](https://ethereum.github.io/execution-apis/api-documentation/)
- [Alchemy 文档](https://docs.alchemy.com/)
- [Infura 文档](https://docs.infura.io/)
- [chainlist.org](https://chainlist.org/) — 各链公共 RPC 列表
