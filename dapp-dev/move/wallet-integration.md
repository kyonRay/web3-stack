# Move DApp 钱包集成（Sui / Aptos）

> **标签：** sui-wallet、aptos-wallet、dapp-kit、wallet-adapter

---

## 1. Sui 钱包集成

### @mysten/dapp-kit（官方推荐）

```bash
npm install @mysten/dapp-kit @mysten/sui @tanstack/react-query
```

```tsx
// providers.tsx
import { SuiClientProvider, WalletProvider } from '@mysten/dapp-kit'
import { getFullnodeUrl } from '@mysten/sui/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import '@mysten/dapp-kit/dist/index.css'

const queryClient = new QueryClient()
const networks = {
    mainnet: { url: getFullnodeUrl('mainnet') },
    testnet: { url: getFullnodeUrl('testnet') },
}

export function SuiProviders({ children }: { children: React.ReactNode }) {
    return (
        <QueryClientProvider client={queryClient}>
            <SuiClientProvider networks={networks} defaultNetwork="mainnet">
                <WalletProvider autoConnect>{children}</WalletProvider>
            </SuiClientProvider>
        </QueryClientProvider>
    )
}
```

```tsx
// 连接钱包 UI
import { ConnectButton } from '@mysten/dapp-kit'

function Header() {
    return (
        <header>
            <ConnectButton />
        </header>
    )
}

// 使用钱包状态
import { useCurrentAccount, useCurrentWallet } from '@mysten/dapp-kit'

function WalletInfo() {
    const account = useCurrentAccount()
    const { isConnected } = useCurrentWallet()

    if (!isConnected) return <div>请连接 Sui 钱包</div>

    return (
        <div>
            <p>地址: {account?.address}</p>
            <p>钱包: {account?.chains.join(', ')}</p>
        </div>
    )
}
```

---

## 2. Sui 主流钱包

| 钱包 | 平台 | 特点 |
|------|------|------|
| **Sui Wallet**（官方） | 浏览器 | 官方，稳定 |
| **Suiet** | 浏览器/移动 | 简洁，支持 dApp Kit |
| **Ethos** | 浏览器/移动 | 社交功能 |
| **Nightly** | 浏览器/移动 | 支持多链 |
| **Ledger** | 硬件 | 安全存储 |

---

## 3. Aptos 钱包集成

### Aptos Wallet Adapter（官方）

```bash
npm install @aptos-labs/wallet-adapter-react @aptos-labs/wallet-adapter-core
# 钱包插件
npm install @aptos-labs/aptos-wallet-adapter petra-plugin-wallet-adapter
```

```tsx
// providers.tsx
import { AptosWalletAdapterProvider } from '@aptos-labs/wallet-adapter-react'
import { PetraWallet } from 'petra-plugin-wallet-adapter'
import { Network } from '@aptos-labs/ts-sdk'

const wallets = [new PetraWallet()]

export function AptosProviders({ children }: { children: React.ReactNode }) {
    return (
        <AptosWalletAdapterProvider
            plugins={wallets}
            autoConnect={true}
            dappConfig={{ network: Network.MAINNET }}
            optInWallets={['Petra', 'Nightly', 'Pontem Wallet', 'Mizu Wallet']}
        >
            {children}
        </AptosWalletAdapterProvider>
    )
}
```

```tsx
import { useWallet } from '@aptos-labs/wallet-adapter-react'

function WalletButton() {
    const { account, connected, connect, disconnect, wallets } = useWallet()

    if (connected) {
        return (
            <div>
                <p>地址: {account?.address}</p>
                <button onClick={disconnect}>断开</button>
            </div>
        )
    }

    return (
        <div>
            {wallets.map(wallet => (
                <button key={wallet.name} onClick={() => connect(wallet.name)}>
                    连接 {wallet.name}
                </button>
            ))}
        </div>
    )
}
```

---

## 4. Aptos 主流钱包

| 钱包 | 平台 | 特点 |
|------|------|------|
| **Petra**（官方） | 浏览器 | 官方，最广泛支持 |
| **Pontem** | 浏览器/移动 | 多功能，DeFi 集成 |
| **Nightly** | 浏览器/移动 | 支持多链 |
| **Martian** | 浏览器 | NFT 管理好 |
| **Ledger** | 硬件 | 安全存储 |

---

## 参考资源

- [Sui dApp Kit 文档](https://sdk.mystenlabs.com/dapp-kit)
- [Aptos Wallet Adapter](https://github.com/aptos-labs/aptos-wallet-adapter)
- [Sui 支持的钱包列表](https://docs.sui.io/guides/developer/getting-started/sui-environment)
