# Solana 钱包集成

> **标签：** phantom、backpack、wallet-adapter、solana

Solana DApp 通过 **Wallet Adapter** 统一接入多种钱包（Phantom、Backpack、Solflare 等），无需为每种钱包单独编写代码。

---

## 1. Wallet Adapter 生态

```
@solana/wallet-adapter-react        # React Hooks（useWallet、useConnection）
@solana/wallet-adapter-react-ui     # 预构建的连接钱包 UI 按钮
@solana/wallet-adapter-wallets      # 各钱包适配器集合
@solana/wallet-adapter-base         # 接口与类型
```

---

## 2. 安装与配置

```bash
npm install \
  @solana/wallet-adapter-react \
  @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-wallets \
  @solana/wallet-adapter-base \
  @solana/web3.js
```

### 根组件配置

```tsx
// providers.tsx
'use client'
import { useMemo } from 'react'
import { ConnectionProvider, WalletProvider } from '@solana/wallet-adapter-react'
import { WalletAdapterNetwork } from '@solana/wallet-adapter-base'
import { WalletModalProvider } from '@solana/wallet-adapter-react-ui'
import {
    PhantomWalletAdapter,
    SolflareWalletAdapter,
    BackpackWalletAdapter,
} from '@solana/wallet-adapter-wallets'
import { clusterApiUrl } from '@solana/web3.js'
import '@solana/wallet-adapter-react-ui/styles.css'

export function SolanaProviders({ children }: { children: React.ReactNode }) {
    const network = WalletAdapterNetwork.Mainnet
    const endpoint = useMemo(() => clusterApiUrl(network), [network])

    const wallets = useMemo(
        () => [
            new PhantomWalletAdapter(),
            new SolflareWalletAdapter(),
            new BackpackWalletAdapter(),
        ],
        [network]
    )

    return (
        <ConnectionProvider endpoint={endpoint}>
            <WalletProvider wallets={wallets} autoConnect>
                <WalletModalProvider>
                    {children}
                </WalletModalProvider>
            </WalletProvider>
        </ConnectionProvider>
    )
}
```

---

## 3. 连接钱包 UI

```tsx
import { WalletMultiButton } from '@solana/wallet-adapter-react-ui'
import { useWallet } from '@solana/wallet-adapter-react'

function WalletSection() {
    const { publicKey, connected, disconnect, signMessage } = useWallet()

    return (
        <div>
            {/* 预构建的钱包按钮（自动处理连接/断开/切换） */}
            <WalletMultiButton />

            {connected && publicKey && (
                <div>
                    <p>已连接: {publicKey.toBase58()}</p>
                    <button onClick={disconnect}>断开连接</button>
                </div>
            )}
        </div>
    )
}
```

---

## 4. 签名与验证

```tsx
import { useWallet } from '@solana/wallet-adapter-react'
import bs58 from 'bs58'
import nacl from 'tweetnacl'

function SignMessage() {
    const { publicKey, signMessage } = useWallet()

    const handleSign = async () => {
        if (!publicKey || !signMessage) return

        const message = new TextEncoder().encode('验证身份: ' + Date.now())
        const signature = await signMessage(message)

        // 验证签名
        const isValid = nacl.sign.detached.verify(
            message,
            signature,
            publicKey.toBytes()
        )
        console.log('签名有效:', isValid)
        console.log('签名:', bs58.encode(signature))
    }

    return <button onClick={handleSign}>签名验证</button>
}
```

---

## 5. 自定义 RPC 端点

```tsx
// 使用 Helius 或 QuickNode 替代公共 RPC
const endpoint = `https://mainnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}`

// 或 devnet 测试
const endpoint = `https://devnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}`

<ConnectionProvider endpoint={endpoint}>
```

---

## 6. 主流 Solana 钱包

| 钱包 | 平台 | 特点 |
|------|------|------|
| **Phantom** | 浏览器/移动 | 最流行，用户友好 |
| **Backpack** | 浏览器/移动 | 支持 xNFT（可执行 NFT） |
| **Solflare** | 浏览器/移动 | 支持 Ledger 硬件钱包 |
| **Ledger** | 硬件 | 安全存储 |
| **Glow** | 浏览器 | 简洁 UI |

---

## 参考资源

- [Wallet Adapter GitHub](https://github.com/solana-labs/wallet-adapter)
- [Solana Wallet Adapter 文档](https://solana-labs.github.io/wallet-adapter/)
- [Helius RPC](https://helius.dev/)
