# Rust for Solana 合约开发

> **标签：** rust、所有权、生命周期、solana、合约开发

Solana 程序（合约）使用 Rust 编写。本文聚焦 Solana 开发所需的 Rust 核心概念，不是 Rust 完整教程。

---

## 1. Solana 开发必备的 Rust 概念

### 1.1 所有权与借用

```rust
// 所有权（每个值只有一个所有者）
let s1 = String::from("hello");
let s2 = s1; // s1 move 到 s2，s1 失效

// 借用（引用，不转移所有权）
let s1 = String::from("hello");
let len = calculate_length(&s1); // 传引用
println!("{}", s1); // s1 仍有效

// 可变借用（同一时刻只能有一个）
let mut s = String::from("hello");
let r = &mut s;
r.push_str(" world");
```

### 1.2 Solana 程序中的关键类型

```rust
use solana_program::{
    account_info::AccountInfo,  // 账户信息（程序接收的账户列表）
    pubkey::Pubkey,             // 32 字节公钥
    program_error::ProgramError, // 错误类型
    entrypoint::ProgramResult,  // Result<(), ProgramError>
};
```

### 1.3 Result 与错误处理

```rust
// Solana 程序广泛使用 Result
fn process(accounts: &[AccountInfo]) -> ProgramResult {
    // ? 运算符：若 Err 则提前返回
    let owner = next_account_info(&mut accounts.iter())?;

    if !owner.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    Ok(())
}
```

### 1.4 Traits（特征）

```rust
// Solana 序列化用 BorshSerialize/BorshDeserialize
use borsh::{BorshDeserialize, BorshSerialize};

#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct CounterAccount {
    pub count: u64,
    pub authority: Pubkey,
}

// 读写账户数据
let counter: CounterAccount = CounterAccount::try_from_slice(&account.data.borrow())?;
counter.serialize(&mut &mut account.data.borrow_mut()[..])?;
```

---

## 2. 原生 Solana 程序结构

```rust
// src/lib.rs
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    pubkey::Pubkey,
    msg,
};

// 声明入口点
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,           // 本程序的地址
    accounts: &[AccountInfo],      // 调用传入的账户列表
    instruction_data: &[u8],       // 指令数据（手动反序列化）
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;

    // 检查账户所有者
    if account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    msg!("Hello, Solana!"); // 类似 console.log，写入交易日志

    Ok(())
}
```

---

## 3. 账户数据与 Borsh 序列化

Solana 账户数据是原始字节，需手动序列化/反序列化：

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::program_error::ProgramError;

#[derive(BorshSerialize, BorshDeserialize, Debug, Clone)]
pub struct VaultState {
    pub is_initialized: bool,
    pub authority: Pubkey,
    pub total_deposits: u64,
}

impl VaultState {
    pub const LEN: usize = 1 + 32 + 8; // bool + Pubkey + u64

    pub fn from_account(account: &AccountInfo) -> Result<Self, ProgramError> {
        Self::try_from_slice(&account.data.borrow())
            .map_err(|_| ProgramError::InvalidAccountData)
    }

    pub fn save(&self, account: &AccountInfo) -> ProgramResult {
        self.serialize(&mut &mut account.data.borrow_mut()[..])
            .map_err(|_| ProgramError::InvalidAccountData)
    }
}
```

---

## 4. CPI（跨程序调用）中的 Rust

```rust
use solana_program::program::invoke;
use spl_token::instruction as token_ix;

// 调用 Token Program 的 transfer
let transfer_ix = token_ix::transfer(
    &spl_token::id(),
    source_account.key,
    destination_account.key,
    authority.key,
    &[],
    amount,
)?;

invoke(
    &transfer_ix,
    &[
        source_account.clone(),
        destination_account.clone(),
        authority.clone(),
        token_program.clone(),
    ],
)?;
```

---

## 5. 与 Anchor 的关系

原生 Solana 程序手动处理账户验证与序列化，代码量大且易错。**Anchor** 用宏大幅简化这些样板代码：

```
原生 Rust：手动检查 is_signer、owner、数据大小、反序列化
Anchor：用 #[derive(Accounts)] 宏声明式地定义验证规则
```

建议路径：理解原生程序结构（有助于调试）→ 实际开发使用 Anchor。

---

## 参考资源

- [The Rust Book](https://doc.rust-lang.org/book/)（官方 Rust 教程）
- [Solana 程序开发（原生）](https://solana.com/developers/guides/getstarted/intro-to-native-rust)
- [Borsh 规范](https://borsh.io/)
