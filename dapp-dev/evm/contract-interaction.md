# EVM 合约交互

> **标签：** ethers.js、web3.js、viem、abi、合约调用、事件监听

前端与链上合约交互需要：**合约地址 + ABI + 库**。现代推荐使用 **Viem**（配合 Wagmi），或 **Ethers.js v6**。

---

## 1. ABI 基础

ABI（Application Binary Interface）是合约与外部世界的接口描述：

```json
[
  {
    "type": "function",
    "name": "transfer",
    "inputs": [
      { "name": "to", "type": "address" },
      { "name": "amount", "type": "uint256" }
    ],
    "outputs": [{ "name": "", "type": "bool" }],
    "stateMutability": "nonpayable"
  },
  {
    "type": "event",
    "name": "Transfer",
    "inputs": [
      { "name": "from", "type": "address", "indexed": true },
      { "name": "to", "type": "address", "indexed": true },
      { "name": "value", "type": "uint256", "indexed": false }
    ]
  }
]
```

---

## 2. Viem（推荐，与 Wagmi 配套）

```typescript
import { createPublicClient, createWalletClient, http, parseAbi, parseEther } from 'viem'
import { mainnet } from 'viem/chains'
import { privateKeyToAccount } from 'viem/accounts'

// 读客户端（无需签名）
const publicClient = createPublicClient({
    chain: mainnet,
    transport: http('https://mainnet.infura.io/v3/YOUR_KEY'),
})

// 写客户端（需要签名者）
const account = privateKeyToAccount('0x...')
const walletClient = createWalletClient({
    account,
    chain: mainnet,
    transport: http(),
})

// 内联 ABI（类型安全）
const erc20Abi = parseAbi([
    'function balanceOf(address) view returns (uint256)',
    'function transfer(address to, uint256 amount) returns (bool)',
    'event Transfer(address indexed from, address indexed to, uint256 value)',
])

// 读取（view 调用）
const balance = await publicClient.readContract({
    address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // USDC
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: ['0x...'],
})
console.log(balance) // BigInt

// 模拟写入（不广播，仅验证）
const { result } = await publicClient.simulateContract({
    address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    abi: erc20Abi,
    functionName: 'transfer',
    args: ['0xRecipient', parseUnits('100', 6)], // USDC 6 decimals
    account,
})

// 实际写入
const hash = await walletClient.writeContract({
    address: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
    abi: erc20Abi,
    functionName: 'transfer',
    args: ['0xRecipient', parseUnits('100', 6)],
})

// 等待确认
const receipt = await publicClient.waitForTransactionReceipt({ hash })
```

---

## 3. Wagmi Hooks（React 最佳实践）

```tsx
import {
    useReadContract, useWriteContract,
    useWaitForTransactionReceipt
} from 'wagmi'
import { parseUnits } from 'viem'

const USDC_ADDRESS = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48'

function TokenBalance({ userAddress }: { userAddress: `0x${string}` }) {
    const { data: balance, isLoading } = useReadContract({
        address: USDC_ADDRESS,
        abi: erc20Abi,
        functionName: 'balanceOf',
        args: [userAddress],
    })

    if (isLoading) return <div>Loading...</div>
    return <div>余额: {formatUnits(balance ?? 0n, 6)} USDC</div>
}

function TransferButton() {
    const { writeContract, data: hash, isPending } = useWriteContract()
    const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash })

    return (
        <div>
            <button
                disabled={isPending}
                onClick={() =>
                    writeContract({
                        address: USDC_ADDRESS,
                        abi: erc20Abi,
                        functionName: 'transfer',
                        args: ['0xRecipient', parseUnits('100', 6)],
                    })
                }
            >
                {isPending ? '签名中...' : '转账 100 USDC'}
            </button>
            {isConfirming && <div>等待确认...</div>}
            {isSuccess && <div>✅ 转账成功!</div>}
        </div>
    )
}
```

---

## 4. Ethers.js v6

```typescript
import { ethers, BrowserProvider, Contract, parseUnits } from 'ethers'

// 连接 MetaMask
const provider = new BrowserProvider(window.ethereum)
const signer = await provider.getSigner()

// 合约实例
const usdcContract = new Contract(USDC_ADDRESS, erc20Abi, signer)

// 读取（返回 BigInt）
const balance = await usdcContract.balanceOf(userAddress)
console.log(ethers.formatUnits(balance, 6)) // "100.0"

// 写入
const tx = await usdcContract.transfer(recipient, parseUnits('100', 6))
const receipt = await tx.wait() // 等待确认
console.log('Gas used:', receipt.gasUsed)
```

---

## 5. 事件监听

```typescript
// Viem：读取历史事件
const logs = await publicClient.getLogs({
    address: USDC_ADDRESS,
    event: parseAbiItem('event Transfer(address indexed from, address indexed to, uint256 value)'),
    fromBlock: 19000000n,
    toBlock: 'latest',
    args: {
        to: userAddress, // 筛选接收方
    },
})

// Viem：实时监听
const unwatch = publicClient.watchEvent({
    address: USDC_ADDRESS,
    event: parseAbiItem('event Transfer(address indexed from, address indexed to, uint256 value)'),
    onLogs: (logs) => {
        logs.forEach(log => {
            console.log(`Transfer: ${log.args.from} → ${log.args.to}: ${log.args.value}`)
        })
    },
})

// 停止监听
unwatch()
```

---

## 6. 多调用（Multicall）

批量读取节省 RPC 请求数：

```typescript
import { multicall } from 'viem/actions'

// 一次 RPC 请求读取多个合约数据
const results = await publicClient.multicall({
    contracts: [
        { address: TOKEN_A, abi: erc20Abi, functionName: 'balanceOf', args: [userAddress] },
        { address: TOKEN_B, abi: erc20Abi, functionName: 'balanceOf', args: [userAddress] },
        { address: TOKEN_C, abi: erc20Abi, functionName: 'balanceOf', args: [userAddress] },
    ],
})
// results = [{result: bigint, status: 'success'}, ...]
```

---

## 参考资源

- [Viem 文档](https://viem.sh/)
- [Wagmi 文档](https://wagmi.sh/)
- [Ethers.js v6 文档](https://docs.ethers.org/v6/)
- [ABI 编码规范](https://docs.soliditylang.org/en/latest/abi-spec.html)
