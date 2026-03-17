# EVM 钱包集成

> **标签：** metamask、walletconnect、rainbowkit、web3modal、wagmi

EVM DApp 的钱包集成是用户进入 Web3 的第一步。现代标配是 **Wagmi + RainbowKit**（或 Web3Modal）。

---

## 1. 核心概念

- **EIP-1193（Provider 标准）**：浏览器钱包（MetaMask 等）注入 `window.ethereum`，提供统一的 `request()` API
- **WalletConnect v2**：通过 URI 或链接让移动钱包连接 DApp，无需浏览器插件
- **聚合 SDK**：RainbowKit、Web3Modal、ConnectKit 等统一「选择钱包」的 UI 与连接逻辑

---

## 2. 现代标配：Wagmi + RainbowKit

### 安装

```bash
npm install wagmi viem @rainbow-me/rainbowkit @tanstack/react-query
```

### 配置

```typescript
// lib/wagmi.ts
import { getDefaultConfig } from '@rainbow-me/rainbowkit'
import { mainnet, arbitrum, optimism, base, polygon } from 'wagmi/chains'

export const wagmiConfig = getDefaultConfig({
    appName: 'My DApp',
    projectId: 'YOUR_WALLETCONNECT_PROJECT_ID', // cloud.walletconnect.com
    chains: [mainnet, arbitrum, optimism, base, polygon],
    ssr: false, // Next.js SSR 设为 true
})
```

### 根组件配置

```tsx
// app/providers.tsx
'use client'
import { RainbowKitProvider } from '@rainbow-me/rainbowkit'
import { WagmiProvider } from 'wagmi'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { wagmiConfig } from '@/lib/wagmi'
import '@rainbow-me/rainbowkit/styles.css'

const queryClient = new QueryClient()

export function Providers({ children }: { children: React.ReactNode }) {
    return (
        <WagmiProvider config={wagmiConfig}>
            <QueryClientProvider client={queryClient}>
                <RainbowKitProvider>{children}</RainbowKitProvider>
            </QueryClientProvider>
        </WagmiProvider>
    )
}
```

### 使用

```tsx
import { ConnectButton } from '@rainbow-me/rainbowkit'
import { useAccount, useChainId, useSwitchChain } from 'wagmi'

function WalletButton() {
    return <ConnectButton />  // 自带完整 UI（连接、切链、显示余额等）
}

function AccountInfo() {
    const { address, isConnected, chain } = useAccount()
    const chainId = useChainId()
    const { switchChain } = useSwitchChain()

    if (!isConnected) return <div>请先连接钱包</div>

    return (
        <div>
            <p>地址: {address}</p>
            <p>当前链: {chain?.name}</p>
            <button onClick={() => switchChain({ chainId: 42161 })}>
                切换到 Arbitrum
            </button>
        </div>
    )
}
```

---

## 3. 原生 window.ethereum（无框架）

```typescript
// 不依赖 Wagmi 的基本钱包连接
async function connectWallet(): Promise<string[]> {
    if (!window.ethereum) throw new Error('请安装 MetaMask')

    const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts',
    }) as string[]

    return accounts
}

// 监听账户切换
window.ethereum.on('accountsChanged', (accounts: string[]) => {
    console.log('账户切换:', accounts[0])
})

// 监听链切换
window.ethereum.on('chainChanged', (chainId: string) => {
    console.log('链切换:', parseInt(chainId, 16))
    window.location.reload() // 通常建议刷新
})

// 切换网络
await window.ethereum.request({
    method: 'wallet_switchEthereumChain',
    params: [{ chainId: '0xa4b1' }], // Arbitrum One
})
```

---

## 4. Web3Modal（by WalletConnect）

替代 RainbowKit 的选择，与 Wagmi v2 深度集成：

```typescript
import { createWeb3Modal } from '@web3modal/wagmi/react'
import { wagmiConfig } from './wagmi'

createWeb3Modal({
    wagmiConfig,
    projectId: 'YOUR_WALLETCONNECT_PROJECT_ID',
    enableAnalytics: true,
})

// 在组件中使用
import { useWeb3Modal } from '@web3modal/wagmi/react'

function ConnectButton() {
    const { open } = useWeb3Modal()
    return <button onClick={() => open()}>连接钱包</button>
}
```

---

## 5. 钱包类型全览

EVM 生态的钱包按形态可分为以下几类：

### 5.1 浏览器插件钱包

| 钱包 | 特点 |
|------|------|
| **MetaMask** | 最广泛使用，支持自定义 RPC 和网络 |
| **Rabby** | 多链友好，交易预模拟，安全提示 |
| **Coinbase Wallet** | Coinbase 生态集成 |
| **OKX Wallet** | OKX 交易所关联，多链支持 |

### 5.2 App 钱包（移动端）

| 钱包 | 特点 |
|------|------|
| **MetaMask Mobile** | 与插件版功能一致 |
| **imToken** | 中国用户广泛使用，多链 |
| **Trust Wallet** | Binance 旗下，支持多链和 DApp 浏览器 |
| **Rainbow** | 界面美观，RainbowKit 集成友好 |

### 5.3 多签钱包

| 钱包 | 特点 |
|------|------|
| **Safe（Gnosis Safe）** | 最主流多签钱包，支持 m-of-n 签名，合约钱包 |
| **Coinbase Smart Wallet** | 基于 ERC-4337 的智能合约钱包 |

### 5.4 硬件钱包

| 钱包 | 特点 |
|------|------|
| **Ledger** | 最广泛使用，Nano S/X 系列，EVM + 多链 |
| **Trezor** | 开源硬件，Model T/One |
| **imKey** | 国产硬件钱包，适配 imToken |
| **KeyStone** | 气隙（Air-gapped）签名，无 USB 连接 |

### 5.5 智能钱包（ERC-4337 账户抽象）

| 钱包 | 特点 |
|------|------|
| **Pimlico / ZeroDev** | AA 基础设施，支持 Paymaster 和 Bundler |
| **Biconomy** | AA SDK，Gas 代付和 Session Key |
| **Safe AA** | Safe 合约 + 4337 兼容层 |

### 钱包支持情况总览

| 钱包 | 浏览器插件 | 移动 | WalletConnect | EIP-4337 |
|------|----------|------|---------------|---------|
| MetaMask | ✅ | ✅ | ✅ | 部分支持 |
| Coinbase Wallet | ✅ | ✅ | ✅ | ✅（Smart Wallet） |
| WalletConnect | — | ✅（多钱包） | ✅（核心） | — |
| Rabby | ✅ | ✅ | ✅ | — |
| Safe（多签） | — | — | ✅ | ✅ |
| Ledger/Trezor | ✅（硬件） | — | ✅ | — |
| imToken | — | ✅ | ✅ | — |
| KeyStone | ✅（气隙） | — | ✅ | — |

---

## 6. 安全注意事项

- **授权范围最小化**：DApp 只应申请所需的权限
- **验证链 ID**：确认用户在期望的链上再执行操作
- **钓鱼防护**：使用 EIP-712 结构化签名（而非裸 `eth_sign`）
- **模拟交易**：在发送前展示交易预期效果（Tenderly Simulate）

---

## 参考资源

- [RainbowKit 文档](https://rainbowkit.com/docs/introduction)
- [Wagmi 文档](https://wagmi.sh/)
- [Web3Modal 文档](https://docs.walletconnect.com/web3modal/about)
- [EIP-1193 标准](https://eips.ethereum.org/EIPS/eip-1193)
