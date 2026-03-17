# Solana 合约交互

> **标签：** solana-web3.js、anchor-client、交易构建、指令

Solana 合约（程序）交互通过 **@solana/web3.js**（或新版 `@solana/kit`）和 Anchor 生成的客户端实现。

---

## 1. @solana/web3.js 基础

```typescript
import {
    Connection, PublicKey, Transaction,
    SystemProgram, sendAndConfirmTransaction,
    Keypair, LAMPORTS_PER_SOL
} from '@solana/web3.js'

const connection = new Connection('https://api.mainnet-beta.solana.com')

// 查询余额
const balance = await connection.getBalance(new PublicKey(address))
console.log('余额:', balance / LAMPORTS_PER_SOL, 'SOL')

// 查询账户数据
const accountInfo = await connection.getAccountInfo(new PublicKey(programAccount))
console.log('数据:', accountInfo?.data)

// 发送 SOL
const tx = new Transaction().add(
    SystemProgram.transfer({
        fromPubkey: senderKeypair.publicKey,
        toPubkey: new PublicKey(recipient),
        lamports: 0.01 * LAMPORTS_PER_SOL,
    })
)
const signature = await sendAndConfirmTransaction(connection, tx, [senderKeypair])
```

---

## 2. Anchor 客户端（推荐）

Anchor 程序编译时自动生成类型安全的客户端：

```typescript
import { Program, AnchorProvider, web3, BN } from '@coral-xyz/anchor'
import { useAnchorWallet, useConnection } from '@solana/wallet-adapter-react'
import { IDL, MyProgram } from './idl/my_program'

function useMyProgram() {
    const { connection } = useConnection()
    const wallet = useAnchorWallet()

    const program = useMemo(() => {
        if (!wallet) return null
        const provider = new AnchorProvider(connection, wallet, { commitment: 'confirmed' })
        return new Program<MyProgram>(IDL, PROGRAM_ID, provider)
    }, [connection, wallet])

    return program
}

// 调用合约函数
async function initialize(program: Program<MyProgram>, initialCount: number) {
    const [counterPDA] = web3.PublicKey.findProgramAddressSync(
        [Buffer.from('counter'), program.provider.publicKey!.toBuffer()],
        program.programId
    )

    const tx = await program.methods
        .initialize(new BN(initialCount))
        .accounts({
            counter: counterPDA,
            authority: program.provider.publicKey!,
            systemProgram: web3.SystemProgram.programId,
        })
        .rpc()

    console.log('交易签名:', tx)
    return tx
}

// 读取账户数据
async function getCounter(program: Program<MyProgram>, counterPDA: web3.PublicKey) {
    const counter = await program.account.counterAccount.fetch(counterPDA)
    console.log('Count:', counter.count.toNumber())
    return counter
}
```

---

## 3. 交易构建与优化

```typescript
import { Transaction, ComputeBudgetProgram, PublicKey } from '@solana/web3.js'

// 添加优先费（高峰期）
const addPriorityFee = ComputeBudgetProgram.setComputeUnitPrice({
    microLamports: 50_000,
})

// 设置 CU 上限（避免浪费）
const setCULimit = ComputeBudgetProgram.setComputeUnitLimit({
    units: 200_000,
})

const tx = new Transaction()
    .add(setCULimit)
    .add(addPriorityFee)
    .add(myInstruction)

// 获取最新 blockhash（交易有效期 ~150 个区块）
const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash()
tx.recentBlockhash = blockhash
tx.feePayer = wallet.publicKey

// 签名并发送
const signedTx = await wallet.signTransaction(tx)
const signature = await connection.sendRawTransaction(signedTx.serialize())

// 等待确认
await connection.confirmTransaction({
    signature,
    blockhash,
    lastValidBlockHeight,
})
```

---

## 4. 处理 Token 操作

```typescript
import {
    getAssociatedTokenAddress,
    createAssociatedTokenAccountInstruction,
    createTransferInstruction,
    TOKEN_PROGRAM_ID,
    ASSOCIATED_TOKEN_PROGRAM_ID,
} from '@solana/spl-token'

async function transferSPLToken(
    connection: Connection,
    payer: Keypair,
    mint: PublicKey,
    source: PublicKey,
    destination: PublicKey,
    amount: bigint
) {
    // 获取/创建接收方 ATA
    const destAta = await getAssociatedTokenAddress(mint, destination)
    const destAtaInfo = await connection.getAccountInfo(destAta)

    const tx = new Transaction()

    // 如果目标 ATA 不存在，先创建
    if (!destAtaInfo) {
        tx.add(
            createAssociatedTokenAccountInstruction(
                payer.publicKey, destAta, destination, mint
            )
        )
    }

    // 获取发送方 ATA
    const sourceAta = await getAssociatedTokenAddress(mint, source)

    // 添加转账指令
    tx.add(
        createTransferInstruction(sourceAta, destAta, source, amount)
    )

    return sendAndConfirmTransaction(connection, tx, [payer])
}
```

---

## 5. 事件监听

```typescript
// 监听新交易（WebSocket）
const subscriptionId = connection.onLogs(
    programId,
    (logs) => {
        console.log('程序日志:', logs.logs)
        console.log('签名:', logs.signature)
    },
    'confirmed'
)

// Anchor 事件监听（IDL 中定义了事件）
program.addEventListener('CounterIncrementedEvent', (event, slot) => {
    console.log('Counter incremented to:', event.newCount.toNumber())
})

// 停止监听
connection.removeOnLogsListener(subscriptionId)
program.removeEventListener(listenerHandle)
```

---

## 6. 错误处理

```typescript
import { AnchorError } from '@coral-xyz/anchor'

try {
    await program.methods.someMethod().rpc()
} catch (err) {
    if (err instanceof AnchorError) {
        console.log('Anchor 错误码:', err.error.errorCode.number)
        console.log('错误信息:', err.error.errorMessage)
        // 对比 IDL 中定义的错误码
    } else if (err.logs) {
        console.log('交易日志:', err.logs)
    }
}
```

---

## 参考资源

- [@solana/web3.js 文档](https://solana-labs.github.io/solana-web3.js/)
- [Anchor 客户端文档](https://www.anchor-lang.com/docs/clients/typescript)
- [@solana/spl-token 文档](https://solana-labs.github.io/solana-program-library/token/js/)
