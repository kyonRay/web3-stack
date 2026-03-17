# Solana DApp 前端框架

> **标签：** react、next.js、solana-hooks、dapp-scaffold

---

## 1. 推荐技术栈

```
React / Next.js
  + @solana/wallet-adapter-react（钱包 Hooks）
  + @solana/wallet-adapter-react-ui（钱包 UI）
  + @coral-xyz/anchor（合约交互）
  + @solana/spl-token（代币操作）
  + Helius / QuickNode（RPC + 增强 API）
  + TailwindCSS（样式）
```

---

## 2. 使用官方脚手架

```bash
# Solana dApp Scaffold（Next.js + Wallet Adapter）
npx create-solana-dapp my-dapp
# 选择：Next.js、Anchor（如有合约）、TailwindCSS

# 或 Anchor 内置的前端模板
anchor init my-project --template react
```

---

## 3. 核心 Hooks 示例

```tsx
'use client'
import { useWallet, useConnection } from '@solana/wallet-adapter-react'
import { LAMPORTS_PER_SOL } from '@solana/web3.js'
import { useState, useEffect } from 'react'

function SolanaAccountInfo() {
    const { publicKey, connected } = useWallet()
    const { connection } = useConnection()
    const [balance, setBalance] = useState<number | null>(null)

    useEffect(() => {
        if (!publicKey) return

        // 获取余额
        connection.getBalance(publicKey).then(setBalance)

        // 监听余额变化
        const subscriptionId = connection.onAccountChange(publicKey, (info) => {
            setBalance(info.lamports)
        })

        return () => { connection.removeAccountChangeListener(subscriptionId) }
    }, [publicKey, connection])

    if (!connected) return <div>请连接 Solana 钱包</div>

    return (
        <div>
            <p>地址: {publicKey?.toBase58()}</p>
            <p>余额: {balance !== null ? (balance / LAMPORTS_PER_SOL).toFixed(4) : '...'} SOL</p>
        </div>
    )
}
```

---

## 4. 完整 DApp 页面示例

```tsx
// app/page.tsx（Solana 计数器 DApp）
'use client'
import { useWallet } from '@solana/wallet-adapter-react'
import { WalletMultiButton } from '@solana/wallet-adapter-react-ui'
import { useMyProgram, useCounter } from '@/hooks/useCounter'

export default function CounterPage() {
    const { connected } = useWallet()
    const program = useMyProgram()
    const { count, isLoading, increment } = useCounter(program)

    return (
        <main className="flex min-h-screen flex-col items-center p-24">
            <h1 className="text-4xl font-bold mb-8">Solana Counter</h1>

            <WalletMultiButton className="mb-8" />

            {connected && (
                <div className="text-center">
                    <p className="text-2xl mb-4">
                        Count: {isLoading ? '...' : count?.toString()}
                    </p>
                    <button
                        className="px-6 py-3 bg-purple-600 text-white rounded-lg"
                        onClick={increment}
                    >
                        Increment
                    </button>
                </div>
            )}
        </main>
    )
}
```

---

## 5. 状态管理

```tsx
// hooks/useCounter.ts
import { useConnection, useWallet } from '@solana/wallet-adapter-react'
import { useCallback, useEffect, useState } from 'react'
import { Program, AnchorProvider, web3, BN } from '@coral-xyz/anchor'
import { IDL, MyProgram } from '../target/types/my_program'

const PROGRAM_ID = new web3.PublicKey('...')

export function useMyProgram() {
    const { connection } = useConnection()
    const wallet = useAnchorWallet()

    return useMemo(() => {
        if (!wallet) return null
        const provider = new AnchorProvider(connection, wallet, {})
        return new Program<MyProgram>(IDL, PROGRAM_ID, provider)
    }, [connection, wallet])
}

export function useCounter(program: Program<MyProgram> | null) {
    const { publicKey } = useWallet()
    const [count, setCount] = useState<number | null>(null)
    const [isLoading, setIsLoading] = useState(false)

    const [counterPDA] = useMemo(() => {
        if (!publicKey) return [null]
        return web3.PublicKey.findProgramAddressSync(
            [Buffer.from('counter'), publicKey.toBuffer()],
            PROGRAM_ID
        )
    }, [publicKey])

    // 读取计数
    useEffect(() => {
        if (!program || !counterPDA) return
        program.account.counterAccount.fetchNullable(counterPDA).then(acc => {
            setCount(acc?.count.toNumber() ?? null)
        })
    }, [program, counterPDA])

    // 自增操作
    const increment = useCallback(async () => {
        if (!program || !counterPDA || !publicKey) return
        setIsLoading(true)
        try {
            await program.methods
                .increment()
                .accounts({ counter: counterPDA, authority: publicKey })
                .rpc()
            const acc = await program.account.counterAccount.fetch(counterPDA)
            setCount(acc.count.toNumber())
        } finally {
            setIsLoading(false)
        }
    }, [program, counterPDA, publicKey])

    return { count, isLoading, increment }
}
```

---

## 参考资源

- [Solana dApp Scaffold](https://github.com/solana-labs/dapp-scaffold)
- [create-solana-dapp](https://github.com/solana-developers/create-solana-dapp)
- [Wallet Adapter 示例](https://github.com/solana-labs/wallet-adapter/tree/master/packages/starter/nextjs-starter)
