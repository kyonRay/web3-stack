# SPL Token 标准

> **标签：** spl-token、token-2022、mint、ata、solana

Solana 的代币标准是 **SPL Token**（Solana Program Library Token），与 EVM 不同的是：代币逻辑由**统一的 Token Program** 处理，而非每个代币各自部署合约。

---

## 1. SPL Token 核心概念

```
Mint Account（铸币账户）
  ├── mint_authority: 有权铸造新代币的地址
  ├── freeze_authority: 有权冻结账户的地址
  ├── supply: 当前总供应量
  └── decimals: 小数位数（如 9 表示 1 = 1_000_000_000 最小单位）

Token Account（代币账户）
  ├── mint: 对应的 Mint 地址
  ├── owner: 账户所有者
  ├── amount: 余额
  └── delegate / delegated_amount: 授权信息

关联代币账户（ATA，Associated Token Account）
  = PDA(owner_pubkey + token_program_id + mint_pubkey)
  每个用户对每种代币有一个「标准」ATA 地址，可预先计算
```

---

## 2. 创建与操作（@solana/spl-token）

```typescript
import {
    createMint, createAssociatedTokenAccount,
    mintTo, transfer, burn, getAssociatedTokenAddress,
    TOKEN_PROGRAM_ID, ASSOCIATED_TOKEN_PROGRAM_ID
} from '@solana/spl-token'
import { Connection, Keypair } from '@solana/web3.js'

const connection = new Connection('https://api.devnet.solana.com')
const payer = Keypair.generate()

// 1. 创建 Mint（代币类型）
const mint = await createMint(
    connection,
    payer,              // 手续费支付者
    payer.publicKey,   // mint authority
    payer.publicKey,   // freeze authority（可 null）
    9                  // decimals
)

// 2. 创建 ATA
const ata = await createAssociatedTokenAccount(
    connection, payer, mint, payer.publicKey
)

// 3. 铸造代币
await mintTo(
    connection, payer, mint, ata, payer, 1_000_000_000n // 1 token
)

// 4. 获取 ATA 地址（预计算，不需要链上查询）
const ataAddress = await getAssociatedTokenAddress(mint, ownerPublicKey)

// 5. 转账
await transfer(connection, payer, sourceAta, destAta, payer, 500_000_000n)
```

---

## 3. Anchor 中使用 SPL Token

```rust
use anchor_spl::token::{Mint, Token, TokenAccount};
use anchor_spl::associated_token::AssociatedToken;

#[derive(Accounts)]
pub struct InitializeToken<'info> {
    // 自动创建 ATA
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    pub system_program: Program<'info, System>,
}
```

---

## 4. Token-2022（新版代币标准）

Token-2022（又称 Token Extensions）是 SPL Token 的升级版，增加了多种扩展：

| 扩展 | 说明 |
|------|------|
| **Transfer Fee** | 每次转账自动扣费 |
| **Non-Transferable** | 灵魂绑定代币（SBT） |
| **Interest-Bearing** | 自动累计利息的代币 |
| **Confidential Transfer** | 零知识证明隐藏转账金额 |
| **Permanent Delegate** | 永久授权（可随时划走余额） |
| **Transfer Hook** | 转账时调用自定义程序（类 ERC-777 hook） |
| **Metadata** | 链上元数据，无需 Metaplex |

```typescript
import { TOKEN_2022_PROGRAM_ID, createMint } from '@solana/spl-token'

// 创建带 Transfer Fee 的 Token-2022 代币
const mint = await createMint(
    connection, payer, authority, null, 9,
    undefined, undefined,
    TOKEN_2022_PROGRAM_ID  // 使用 Token-2022 程序
)
```

---

## 5. NFT 与 Metaplex

Solana NFT 基于 SPL Token（supply = 1, decimals = 0）+ Metaplex 元数据：

```
NFT = Mint Account（supply=1, decimals=0）
    + Metadata Account（Metaplex Token Metadata Program）
      ├── name, symbol, uri
      ├── creators（版税分配）
      └── collection（系列归属）
```

```typescript
import { createNft } from '@metaplex-foundation/mpl-token-metadata'
import { createUmi } from '@metaplex-foundation/umi-bundle-defaults'

const umi = createUmi('https://api.devnet.solana.com').use(mplTokenMetadata())

const { nft } = await createNft(umi, {
    name: 'My NFT',
    symbol: 'MNFT',
    uri: 'https://arweave.net/metadata.json',
    sellerFeeBasisPoints: percentAmount(5), // 5% 版税
    isCollection: false,
})
```

---

## 参考资源

- [SPL Token 文档](https://spl.solana.com/token)
- [Token-2022 文档](https://spl.solana.com/token-2022)
- [@solana/spl-token](https://www.npmjs.com/package/@solana/spl-token)
- [Metaplex 文档](https://developers.metaplex.com/)
