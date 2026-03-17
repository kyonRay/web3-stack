# EVM DApp 前端框架

> **标签：** react、next.js、wagmi、vue、tailwind

DApp 前端常用 **React + Next.js + Wagmi + RainbowKit**，这是目前最主流的技术栈。

---

## 1. 推荐技术栈

### 现代标配（2024-2025）

```
Next.js 14+ (App Router)
├── Wagmi v2（链与钱包 Hooks）
├── Viem（底层 EVM 交互）
├── RainbowKit / Web3Modal（钱包 UI）
├── TanStack Query（异步状态）
└── TailwindCSS + shadcn/ui（UI）
```

### 快速创建项目

```bash
# create-wagmi（官方脚手架）
npm create wagmi@latest my-dapp
# 选择：Next.js + RainbowKit

# 或 create-next-app + 手动安装
npx create-next-app@latest my-dapp --typescript --tailwind --app
cd my-dapp
npm install wagmi viem @rainbow-me/rainbowkit @tanstack/react-query
```

---

## 2. Next.js App Router 集成

```typescript
// app/layout.tsx
import { Providers } from '@/components/providers'
import '@rainbow-me/rainbowkit/styles.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
    return (
        <html lang="zh">
            <body>
                <Providers>{children}</Providers>
            </body>
        </html>
    )
}
```

```typescript
// components/providers.tsx（客户端组件）
'use client'
import { getDefaultConfig, RainbowKitProvider } from '@rainbow-me/rainbowkit'
import { WagmiProvider } from 'wagmi'
import { mainnet, arbitrum } from 'wagmi/chains'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const config = getDefaultConfig({
    appName: 'My DApp',
    projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID!,
    chains: [mainnet, arbitrum],
    ssr: true,  // Next.js SSR 模式
})

const queryClient = new QueryClient()

export function Providers({ children }: { children: React.ReactNode }) {
    return (
        <WagmiProvider config={config}>
            <QueryClientProvider client={queryClient}>
                <RainbowKitProvider>{children}</RainbowKitProvider>
            </QueryClientProvider>
        </WagmiProvider>
    )
}
```

### 服务端与客户端组件

```tsx
// app/page.tsx（服务端组件，可 SSR）
import { TokenBalance } from '@/components/token-balance' // 客户端组件

export default function Home() {
    return (
        <main>
            <h1>我的 DApp</h1>
            <TokenBalance />  {/* 标记为 'use client' */}
        </main>
    )
}
```

---

## 3. 核心 Wagmi Hooks

```tsx
'use client'
import {
    useAccount,             // 账户信息
    useBalance,             // ETH 余额
    useReadContract,        // 读合约
    useWriteContract,       // 写合约
    useWaitForTransactionReceipt, // 等待确认
    useChainId,             // 当前链 ID
    useSwitchChain,         // 切链
    useWatchContractEvent,  // 监听事件
} from 'wagmi'

function DAppUI() {
    const { address, isConnected } = useAccount()
    const { data: balance } = useBalance({ address })
    const chainId = useChainId()

    // 读合约（自动缓存 + 轮询）
    const { data: tokenBalance } = useReadContract({
        address: TOKEN_ADDRESS,
        abi: erc20Abi,
        functionName: 'balanceOf',
        args: [address!],
        query: { enabled: !!address },
    })

    // 写合约
    const { writeContract, data: txHash, isPending } = useWriteContract()
    const { isLoading: isTxPending } = useWaitForTransactionReceipt({
        hash: txHash,
    })

    return (
        <div>
            {isConnected ? (
                <>
                    <p>地址: {address}</p>
                    <p>ETH: {balance?.formatted} {balance?.symbol}</p>
                    <p>链 ID: {chainId}</p>
                    <button
                        disabled={isPending || isTxPending}
                        onClick={() => writeContract({ /* ... */ })}
                    >
                        {isPending ? '签名...' : isTxPending ? '确认中...' : '操作'}
                    </button>
                </>
            ) : (
                <ConnectButton />
            )}
        </div>
    )
}
```

---

## 4. Vue 生态

```typescript
// Vue + @wagmi/vue（Beta）
import { WagmiPlugin, createConfig, http } from '@wagmi/vue'
import { mainnet } from 'viem/chains'
import { createApp } from 'vue'
import App from './App.vue'

const config = createConfig({
    chains: [mainnet],
    transports: { [mainnet.id]: http() },
})

createApp(App).use(WagmiPlugin, { config }).mount('#app')
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { useAccount, useReadContract } from '@wagmi/vue'

const { address, isConnected } = useAccount()
const { data: balance } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [address.value!],
})
</script>
```

---

## 5. 交易状态管理

```tsx
// 完整的交易流程状态管理示例
function MintButton() {
    const { writeContract, data: hash, isPending, error } = useWriteContract()
    const { isLoading: isConfirming, isSuccess, data: receipt } =
        useWaitForTransactionReceipt({ hash })

    const states = {
        idle: !isPending && !isConfirming && !isSuccess,
        pending: isPending,
        confirming: isConfirming,
        success: isSuccess,
    }

    return (
        <div>
            <button
                disabled={states.pending || states.confirming}
                onClick={() =>
                    writeContract({
                        address: NFT_ADDRESS,
                        abi: nftAbi,
                        functionName: 'mint',
                        args: [1n],
                        value: parseEther('0.01'),
                    })
                }
            >
                {states.pending && '签名中...'}
                {states.confirming && '链上确认中...'}
                {states.success && '✅ Mint 成功!'}
                {states.idle && 'Mint NFT'}
            </button>

            {hash && (
                <a href={`https://etherscan.io/tx/${hash}`} target="_blank">
                    查看交易
                </a>
            )}

            {error && <p className="text-red-500">错误: {error.message}</p>}

            {isSuccess && receipt && (
                <p>Gas 消耗: {receipt.gasUsed.toString()}</p>
            )}
        </div>
    )
}
```

---

## 参考资源

- [Wagmi 文档](https://wagmi.sh/)
- [RainbowKit 文档](https://rainbowkit.com/)
- [Next.js 文档](https://nextjs.org/docs)
- [Viem 文档](https://viem.sh/)
- [shadcn/ui](https://ui.shadcn.com/)
