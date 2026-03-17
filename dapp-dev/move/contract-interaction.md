# Move DApp 合约交互（Sui / Aptos）

> **标签：** sui-sdk、aptos-sdk、ptb、move-call

---

## 1. Sui 合约交互

### @mysten/sui 核心库

```typescript
import { SuiClient, getFullnodeUrl } from '@mysten/sui/client'
import { Transaction } from '@mysten/sui/transactions'

const client = new SuiClient({ url: getFullnodeUrl('mainnet') })

// 读取链上数据
const object = await client.getObject({
    id: '0x1234...objectId',
    options: { showContent: true },
})
console.log('对象数据:', object.data?.content)

// 查询动态字段
const fields = await client.getDynamicFields({ parentId: '0xParent...' })
```

### 调用合约（PTB）

```typescript
import { useSignAndExecuteTransaction } from '@mysten/dapp-kit'
import { Transaction } from '@mysten/sui/transactions'

function CounterButton({ counterId }: { counterId: string }) {
    const { mutate: signAndExecute } = useSignAndExecuteTransaction()

    const handleIncrement = () => {
        const tx = new Transaction()

        tx.moveCall({
            target: `${PACKAGE_ID}::counter::increment`,
            arguments: [tx.object(counterId)],
        })

        signAndExecute(
            { transaction: tx },
            {
                onSuccess: (result) => {
                    console.log('交易成功:', result.digest)
                },
                onError: (error) => {
                    console.error('交易失败:', error)
                },
            }
        )
    }

    return <button onClick={handleIncrement}>Increment</button>
}
```

### PTB 复杂组合

```typescript
// PTB：一次交易完成多步操作
const tx = new Transaction()

// 1. 分割 SUI
const [payment] = tx.splitCoins(tx.gas, [1_000_000_000n]) // 1 SUI

// 2. 用 SUI 购买 NFT（合约返回 NFT 对象）
const [nft] = tx.moveCall({
    target: `${MARKETPLACE_PACKAGE}::marketplace::buy`,
    arguments: [
        tx.object(LISTING_ID),
        payment,
    ],
})

// 3. 将 NFT 转入 vault（使用上一步返回的 NFT）
tx.moveCall({
    target: `${VAULT_PACKAGE}::vault::deposit`,
    arguments: [tx.object(VAULT_ID), nft],
})

// 整个流程原子执行
```

### 查询 Sui 索引器

```typescript
// 查询账户持有的 NFT
const nfts = await client.getOwnedObjects({
    owner: walletAddress,
    filter: { StructType: `${NFT_PACKAGE}::nft::NFT` },
    options: { showContent: true, showDisplay: true },
})

// 订阅事件（实时）
const unsubscribe = await client.subscribeEvent({
    filter: { Package: PACKAGE_ID },
    onMessage: (event) => {
        console.log('事件:', event)
    },
})
```

---

## 2. Aptos 合约交互

### @aptos-labs/ts-sdk

```typescript
import { Aptos, AptosConfig, Network, InputViewFunctionData } from '@aptos-labs/ts-sdk'

const config = new AptosConfig({ network: Network.MAINNET })
const aptos = new Aptos(config)

// 读取链上数据（view 函数）
const result = await aptos.view({
    payload: {
        function: `${MODULE_ADDRESS}::counter::get_count`,
        typeArguments: [],
        functionArguments: [accountAddress],
    },
} satisfies InputViewFunctionData)
console.log('Count:', result[0])
```

### 构建并发送交易

```typescript
import { useWallet } from '@aptos-labs/wallet-adapter-react'
import { Aptos, AptosConfig, Network } from '@aptos-labs/ts-sdk'

function IncrementButton({ ownerAddress }: { ownerAddress: string }) {
    const { signAndSubmitTransaction } = useWallet()
    const aptos = new Aptos(new AptosConfig({ network: Network.MAINNET }))

    const handleIncrement = async () => {
        const response = await signAndSubmitTransaction({
            data: {
                function: `${MODULE_ADDRESS}::counter::increment`,
                typeArguments: [],
                functionArguments: [ownerAddress], // entry 函数参数
            },
        })

        // 等待确认
        const committedTx = await aptos.waitForTransaction({
            transactionHash: response.hash,
        })
        console.log('确认:', committedTx.success)
    }

    return <button onClick={handleIncrement}>Increment</button>
}
```

### 多步操作（Batch）

```typescript
// Aptos 不支持 PTB，但可以用 multi-agent transaction 或顺序调用
// 对于需要原子性的多步操作，需要在合约层面组合
const tx = await aptos.transaction.build.simple({
    sender: senderAddress,
    data: {
        function: `${MODULE}::vault::deposit_and_stake`,
        typeArguments: ['0x1::aptos_coin::AptosCoin'],
        functionArguments: [amount],
    },
})
```

---

## 参考资源

- [Sui TypeScript SDK](https://sdk.mystenlabs.com/typescript)
- [Sui dApp Kit Hooks](https://sdk.mystenlabs.com/dapp-kit)
- [Aptos TypeScript SDK](https://aptos.dev/sdks/ts-sdk/)
