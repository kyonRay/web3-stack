# PDA 与 CPI

> **标签：** pda、cpi、程序派生地址、跨程序调用

PDA（Program Derived Address）和 CPI（Cross-Program Invocation）是 Solana 程序设计的两大核心机制，必须深入理解。

---

## 1. PDA（程序派生地址）

### 为什么需要 PDA

EVM 中，合约本身有地址，可以持有资产。Solana 中，程序是无状态的，需要通过 PDA 创建**程序控制的账户**（只有对应程序可以"签名"操作该账户）。

### PDA 的关键属性

- 由 `(seeds, program_id)` 确定性推导，不在 ed25519 椭圆曲线上（因此没有私钥）
- 只有对应 `program_id` 的程序可以为该 PDA 签名（CPI with signer seeds）
- 任何人都可以预计算 PDA 地址

### PDA 推导

```typescript
import { PublicKey } from '@solana/web3.js'

const [pda, bump] = PublicKey.findProgramAddressSync(
    [
        Buffer.from("vault"),
        userPublicKey.toBuffer(),
        mintPublicKey.toBuffer(),
    ],
    programId
)
// bump 是保证 PDA 不在曲线上的最小值（0-255）
```

```rust
// Rust 中推导
use solana_program::pubkey::Pubkey;

let (pda, bump) = Pubkey::find_program_address(
    &[b"vault", user.key.as_ref(), mint.key.as_ref()],
    program_id,
);
```

### Anchor 中的 PDA

```rust
// 声明时定义 seeds 和 bump
#[account(
    seeds = [b"vault", user.key().as_ref(), mint.key().as_ref()],
    bump,                           // Anchor 自动验证 bump
    has_one = user,                // vault.user == user.key()
)]
pub vault: Account<'info, VaultState>,

// 存储 bump 避免重复计算
#[account]
pub struct VaultState {
    pub user: Pubkey,
    pub mint: Pubkey,
    pub bump: u8,
    pub amount: u64,
}
```

---

## 2. CPI（跨程序调用）

### 基本 CPI（用 Signer 签名）

```rust
use solana_program::program::invoke;

// 调用 System Program 创建账户
let create_account_ix = system_instruction::create_account(
    payer.key,          // 手续费支付者
    new_account.key,    // 新账户地址
    lamports,           // 租金（从 payer 转出）
    space as u64,       // 数据大小（字节）
    program_id,         // 新账户的 owner
);

invoke(
    &create_account_ix,
    &[payer.clone(), new_account.clone()],
)?;
```

### PDA 签名 CPI（invoke_signed）

当 CPI 需要 PDA 签名时（只有 PDA 对应的程序可以签名）：

```rust
use solana_program::program::invoke_signed;

// PDA seeds（需要与 PDA 推导时完全一致）
let seeds = &[b"vault", user.key.as_ref(), &[vault_bump]];

invoke_signed(
    &transfer_instruction,
    &[vault_account.clone(), destination.clone(), token_program.clone()],
    &[seeds], // 提供 seeds，让运行时验证 PDA 签名
)?;
```

### Anchor 的 CPI 封装

```rust
use anchor_spl::token::{self, Transfer, Token};

// 普通账户签名
let cpi_ctx = CpiContext::new(
    ctx.accounts.token_program.to_account_info(),
    Transfer {
        from: ctx.accounts.user_token.to_account_info(),
        to: ctx.accounts.vault_token.to_account_info(),
        authority: ctx.accounts.user.to_account_info(),
    },
);
token::transfer(cpi_ctx, amount)?;

// PDA 签名
let seeds = &[b"vault", ctx.accounts.user.key.as_ref(), &[ctx.accounts.vault.bump]];
let signer_seeds = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    ctx.accounts.token_program.to_account_info(),
    Transfer {
        from: ctx.accounts.vault_token.to_account_info(),
        to: ctx.accounts.user_token.to_account_info(),
        authority: ctx.accounts.vault.to_account_info(), // PDA 作为 authority
    },
    signer_seeds,
);
token::transfer(cpi_ctx, amount)?;
```

---

## 3. 实际案例：质押合约

结合 PDA + CPI 的典型模式：

```rust
// 质押：用户将代币转入程序控制的 PDA vault
pub fn stake(ctx: Context<Stake>, amount: u64) -> Result<()> {
    // CPI：从用户 ATA 转到 vault ATA（vault PDA 是 authority）
    let cpi_ctx = CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.user_ata.to_account_info(),
            to: ctx.accounts.vault_ata.to_account_info(),
            authority: ctx.accounts.user.to_account_info(),
        },
    );
    token::transfer(cpi_ctx, amount)?;

    // 更新质押状态
    let stake_info = &mut ctx.accounts.stake_info;
    stake_info.amount = stake_info.amount.checked_add(amount).unwrap();
    stake_info.staked_at = Clock::get()?.unix_timestamp;
    Ok(())
}

// 解押：程序（通过 PDA 签名）将代币返还用户
pub fn unstake(ctx: Context<Unstake>) -> Result<()> {
    let amount = ctx.accounts.stake_info.amount;
    let seeds = &[b"vault", ctx.accounts.user.key.as_ref(), &[ctx.accounts.vault.bump]];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.vault_ata.to_account_info(),
            to: ctx.accounts.user_ata.to_account_info(),
            authority: ctx.accounts.vault.to_account_info(),
        },
        &[seeds],
    );
    token::transfer(cpi_ctx, amount)?;

    // 关闭质押账户，返还租金
    ctx.accounts.stake_info.close(ctx.accounts.user.to_account_info())?;
    Ok(())
}
```

---

## 4. 注意事项

- **Reentrancy（重入）**：Solana 禁止同一程序在调用栈中出现两次，从语言层面防止重入
- **AccountInfo 克隆**：CPI 传入账户时需要 `.clone()`，因为 `AccountInfo` 内含 `RefCell`
- **Lamport 检查**：创建账户前确认租金豁免金额（`Rent::get()?.minimum_balance(space)`）
- **账户顺序**：accounts 数组顺序必须与程序期望的一致

---

## 参考资源

- [Solana 程序 PDA 文档](https://docs.solana.com/developing/programming-model/calling-between-programs)
- [Anchor CPI 文档](https://www.anchor-lang.com/docs/cross-program-invocations)
- [Solana Cookbook — PDAs](https://solanacookbook.com/core-concepts/pdas.html)
