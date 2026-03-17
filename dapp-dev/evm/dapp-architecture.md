# DApp 架构设计

> **标签：** dapp-架构、前端、rpc、智能合约、去中心化存储、ipfs、arweave

DApp（去中心化应用）由**前端**、**智能合约**、**数据层**三部分构成，各层的去中心化程度决定了整体的抗审查能力。

---

## 1. DApp 架构全景

```
用户浏览器
    │
    ├── 前端（UI 层）
    │     ├── React / Next.js
    │     ├── Wagmi + Viem（链交互）
    │     └── RainbowKit / Web3Modal（钱包 UI）
    │
    ├── 数据层
    │     ├── RPC 节点（读链上状态）
    │     │     └── Alchemy / Infura / QuickNode / 自建节点
    │     ├── 索引层（历史数据）
    │     │     └── The Graph / Dune / Alchemy NFT API
    │     └── 去中心化存储（文件/元数据）
    │           └── IPFS / Arweave / EthStorage
    │
    └── 链上层
          ├── 智能合约（业务逻辑）
          ├── 代理合约（可升级）
          └── 多签 / Timelock（权限管理）
```

---

## 2. 前端 ↔ 链端交互模型

### 2.1 读操作（View/Pure）

读操作不消耗 Gas，通过 RPC 节点调用：

```typescript
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'

const client = createPublicClient({
  chain: mainnet,
  transport: http('https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY'),
})

// 读合约状态
const balance = await client.readContract({
  address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: [userAddress],
})
```

### 2.2 写操作（State-Changing）

写操作需要钱包签名，消耗 Gas：

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi'

function TransferButton() {
  const { writeContract, data: hash, isPending } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash })

  return (
    <button
      onClick={() => writeContract({
        address: CONTRACT_ADDRESS,
        abi,
        functionName: 'transfer',
        args: [recipient, amount],
      })}
      disabled={isPending || isConfirming}
    >
      {isPending ? '签名中...' : isConfirming ? '确认中...' : '转账'}
    </button>
  )
}
```

### 2.3 事件监听（实时数据）

```typescript
// 监听合约事件（WebSocket）
const unwatch = client.watchContractEvent({
  address: CONTRACT_ADDRESS,
  abi,
  eventName: 'Transfer',
  onLogs: (logs) => {
    logs.forEach(log => console.log('Transfer:', log.args))
  },
})

// 清理
onUnmount(() => unwatch())
```

---

## 3. RPC 节点选型与容错

生产环境应配置多 RPC 节点，避免单点故障：

```typescript
import { fallback, http } from 'viem'

const transport = fallback([
  http('https://eth-mainnet.g.alchemy.com/v2/KEY1'),
  http('https://mainnet.infura.io/v3/KEY2'),
  http('https://rpc.ankr.com/eth'),
])
```

### RPC 节点服务对比

| 服务 | 免费额度 | 特色功能 |
|------|---------|---------|
| **Alchemy** | 300M CU/月 | NFT API、Webhooks、模拟交易 |
| **Infura** | 100K req/天 | 历史悠久，多链支持 |
| **QuickNode** | 500K req/月 | 极速节点，流量分析 |
| **Ankr** | 500 ANKR 抵押免费 | 去中心化 RPC 网络 |

---

## 4. 代理合约模式（可升级架构）

生产 DApp 通常使用代理合约分离逻辑与存储：

```
用户交易
    ↓
Proxy 合约（不变，存储状态）
    ↓ DELEGATECALL
Logic 合约 V1 → Logic 合约 V2（升级后）
```

```typescript
// 前端通过代理合约地址交互（逻辑合约升级时 ABI 可能变化）
const PROXY_ADDRESS = '0x...' // 永远不变
const LOGIC_ABI_V2 = [...] // 升级后更新 ABI

const result = await client.readContract({
  address: PROXY_ADDRESS,
  abi: LOGIC_ABI_V2, // 用最新 ABI 与代理地址交互
  functionName: 'newFunctionInV2',
})
```

---

## 5. 去中心化存储集成

### 5.1 IPFS（内容寻址存储）

```typescript
// 上传到 IPFS（通过 Pinata）
async function uploadToIPFS(file: File): Promise<string> {
  const formData = new FormData()
  formData.append('file', file)

  const res = await fetch('https://api.pinata.cloud/pinning/pinFileToIPFS', {
    method: 'POST',
    headers: { Authorization: `Bearer ${PINATA_JWT}` },
    body: formData,
  })
  const { IpfsHash } = await res.json()
  return `ipfs://${IpfsHash}` // 或 https://gateway.pinata.cloud/ipfs/${IpfsHash}
}
```

### 5.2 Arweave（永久存储）

```typescript
// 通过 Irys（Arweave bundler）上传
import Irys from '@irys/sdk'

const irys = new Irys({
  url: 'https://turbo.ardrive.io',
  token: 'ethereum',
  key: signer, // Ethereum 私钥
})

const receipt = await irys.uploadFile('./image.png')
const permanentUrl = `https://arweave.net/${receipt.id}`
```

### 5.3 EthStorage（基于以太坊的去中心化存储）

EthStorage 将存储数据绑定到以太坊安全性，通过 Blob 扩展：
- 数据永久存储，不依赖 Pinata 等中心化服务
- 通过 ERC-5018 标准访问大型文件
- 适合 DApp 前端资产去中心化托管

```
合约调用 EthStorage 协议 → 数据以 Blob 形式写入以太坊
前端通过 eth_getStorageAt 读取
```

### 5.4 存储方案对比

| 方案 | 去中心化程度 | 费用 | 持久性 | 适合场景 |
|------|------------|------|--------|---------|
| IPFS + Pinata | 中 | 订阅 | 需续费 | NFT 元数据、DApp 资源 |
| Arweave | 高 | 一次性 AR | 永久 | 重要数据永久存储 |
| EthStorage | 高 | ETH | 永久 | DApp 前端去中心化托管 |
| Filecoin | 高 | FIL | 合约期 | 大文件长期存储 |

---

## 6. 前端安全最佳实践

```typescript
// 1. 始终验证链 ID
const { chain } = useAccount()
if (chain?.id !== mainnet.id) {
  return <div>请切换到以太坊主网</div>
}

// 2. 显示交易预期效果（使用 Tenderly Simulation API）
async function simulateTx(tx: TransactionRequest) {
  const result = await tenderly.simulate({
    network_id: '1',
    from: tx.from,
    to: tx.to,
    input: tx.data,
    value: tx.value,
  })
  return result.transaction.transaction_info
}

// 3. 使用 EIP-712 结构化签名（避免裸签名）
const signature = await signTypedData({
  domain: { name: 'MyDApp', version: '1', chainId: 1, verifyingContract: ADDRESS },
  types: { Order: [{ name: 'amount', type: 'uint256' }] },
  primaryType: 'Order',
  message: { amount: 1000n },
})
```

---

## 参考资源

- [Viem 文档](https://viem.sh/)
- [Wagmi 文档](https://wagmi.sh/)
- [Alchemy Docs](https://docs.alchemy.com/)
- [IPFS 文档](https://docs.ipfs.tech/)
- [Arweave Wiki](https://arwiki.wiki/)
- [EthStorage 文档](https://docs.ethstorage.io/)
- [Tenderly Simulation API](https://docs.tenderly.co/simulations-and-forks/simulation-api)
