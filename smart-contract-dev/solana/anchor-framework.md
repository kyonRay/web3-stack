# Anchor 框架

> **标签：** anchor、solana、账户约束、idl、宏

Anchor 是 Solana 最主流的合约开发框架，通过宏系统将大量样板代码（账户验证、序列化、IDL 生成）自动化，让开发者聚焦业务逻辑。

---

## 1. 安装与初始化

```bash
# 安装 Anchor CLI
cargo install --git https://github.com/coral-xyz/anchor avm --locked
avm install latest && avm use latest

# 新建项目
anchor init my-program
cd my-program

# 项目结构
my-program/
├── programs/my-program/src/lib.rs  # 程序逻辑
├── tests/my-program.ts             # TypeScript 测试
├── Anchor.toml                     # 配置
└── target/idl/my-program.json      # 自动生成的 IDL
```

---

## 2. 基础程序结构

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, initial_count: u64) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = initial_count;
        counter.authority = ctx.accounts.authority.key();
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = counter.count.checked_add(1).ok_or(ErrorCode::Overflow)?;
        Ok(())
    }
}

// 账户约束（Anchor 自动验证）
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,                       // 初始化账户
        payer = authority,          // 租金由 authority 支付
        space = 8 + CounterAccount::INIT_SPACE,  // 8 = discriminator
    )]
    pub counter: Account<'info, CounterAccount>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        has_one = authority,        // counter.authority == authority.key()
    )]
    pub counter: Account<'info, CounterAccount>,
    pub authority: Signer<'info>,
}

// 账户数据结构
#[account]
#[derive(InitSpace)]
pub struct CounterAccount {
    pub count: u64,
    pub authority: Pubkey,
}

// 自定义错误
#[error_code]
pub enum ErrorCode {
    #[msg("Counter overflow")]
    Overflow,
}
```

---

## 3. 账户约束（Constraints）

Anchor 的 `#[account(...)]` 提供丰富的约束，无需手动验证：

| 约束 | 说明 |
|------|------|
| `init` | 初始化账户（调用 system_program::create_account） |
| `init_if_needed` | 如果不存在则初始化 |
| `mut` | 标记账户可变（需在 transaction 中声明） |
| `has_one = field` | `account.field == field_account.key()` |
| `constraint = expr` | 自定义布尔约束 |
| `close = target` | 关闭账户，租金返还给 target |
| `seeds = [...]` | PDA 地址派生 |
| `bump` | PDA 的 bump seed |
| `address = pubkey` | 要求账户地址等于指定值 |
| `owner = program` | 要求账户 owner 是指定程序 |

---

## 4. PDA（程序派生地址）与 Anchor

```rust
// 使用 PDA 作为程序控制账户
#[derive(Accounts)]
#[instruction(seed_data: String)]
pub struct CreateUserAccount<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + UserAccount::INIT_SPACE,
        seeds = [b"user", user.key().as_ref(), seed_data.as_bytes()],
        bump,
    )]
    pub user_account: Account<'info, UserAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

// 存储 bump 到账户（避免后续重新计算）
#[account]
#[derive(InitSpace)]
pub struct UserAccount {
    pub user: Pubkey,
    pub bump: u8,
    pub data: u64,
}

// 初始化时存储 bump
pub fn create_user(ctx: Context<CreateUserAccount>, seed_data: String, data: u64) -> Result<()> {
    let account = &mut ctx.accounts.user_account;
    account.user = ctx.accounts.user.key();
    account.bump = ctx.bumps.user_account; // Anchor 自动提供 bump
    account.data = data;
    Ok(())
}
```

---

## 5. CPI（跨程序调用）

```rust
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}

pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    let cpi_accounts = Transfer {
        from: ctx.accounts.from.to_account_info(),
        to: ctx.accounts.to.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    };
    let cpi_ctx = CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_accounts);
    token::transfer(cpi_ctx, amount)?;
    Ok(())
}

// PDA 签名 CPI（程序控制账户）
pub fn transfer_from_vault(ctx: Context<TransferFromVault>, amount: u64) -> Result<()> {
    let seeds = &[b"vault", ctx.accounts.vault.owner.as_ref(), &[ctx.accounts.vault.bump]];
    let signer_seeds = &[&seeds[..]];

    let cpi_ctx = CpiContext::new_with_signer(
        ctx.accounts.token_program.to_account_info(),
        Transfer { /* ... */ },
        signer_seeds,
    );
    token::transfer(cpi_ctx, amount)?;
    Ok(())
}
```

---

## 6. IDL 与客户端

Anchor 自动生成 **IDL（Interface Definition Language）**（类似 ABI），供客户端使用：

```typescript
// TypeScript 客户端（Anchor 生成类型）
import { Program, AnchorProvider, web3 } from '@coral-xyz/anchor'
import { MyProgram, IDL } from '../target/types/my_program'

const provider = AnchorProvider.env()
const program = new Program<MyProgram>(IDL, PROGRAM_ID, provider)

// 调用 initialize
await program.methods
  .initialize(new BN(0))
  .accounts({
    counter: counterPDA,
    authority: wallet.publicKey,
    systemProgram: web3.SystemProgram.programId,
  })
  .rpc()

// 读取账户
const counter = await program.account.counterAccount.fetch(counterPDA)
console.log('Count:', counter.count.toNumber())
```

---

## 参考资源

- [Anchor 官方文档](https://www.anchor-lang.com/)
- [Anchor GitHub](https://github.com/coral-xyz/anchor)
- [Anchor By Example](https://examples.anchor-lang.com/)
- [Solana Cookbook](https://solanacookbook.com/)
