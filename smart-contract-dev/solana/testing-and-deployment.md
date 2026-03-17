# Solana 合约测试与部署

> **标签：** anchor测试、localnet、devnet、程序部署、升级

---

## 1. Anchor 测试框架

Anchor 使用 **Mocha + Chai** 写测试，通过 `@coral-xyz/anchor` 客户端与本地节点交互。

### 基本测试结构

```typescript
// tests/my-program.ts
import * as anchor from '@coral-xyz/anchor'
import { Program } from '@coral-xyz/anchor'
import { MyProgram } from '../target/types/my_program'
import { assert } from 'chai'

describe('my-program', () => {
    const provider = anchor.AnchorProvider.env()
    anchor.setProvider(provider)

    const program = anchor.workspace.MyProgram as Program<MyProgram>
    const wallet = provider.wallet as anchor.Wallet

    let counterPDA: anchor.web3.PublicKey
    let counterBump: number

    before(async () => {
        // 计算 PDA
        ;[counterPDA, counterBump] = anchor.web3.PublicKey.findProgramAddressSync(
            [Buffer.from('counter'), wallet.publicKey.toBuffer()],
            program.programId
        )
    })

    it('Initialize counter', async () => {
        await program.methods
            .initialize(new anchor.BN(0))
            .accounts({
                counter: counterPDA,
                authority: wallet.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
            })
            .rpc()

        const account = await program.account.counterAccount.fetch(counterPDA)
        assert.equal(account.count.toNumber(), 0)
        assert.deepEqual(account.authority, wallet.publicKey)
    })

    it('Increment counter', async () => {
        await program.methods
            .increment()
            .accounts({ counter: counterPDA, authority: wallet.publicKey })
            .rpc()

        const account = await program.account.counterAccount.fetch(counterPDA)
        assert.equal(account.count.toNumber(), 1)
    })

    it('Should fail with wrong authority', async () => {
        const badActor = anchor.web3.Keypair.generate()
        try {
            await program.methods
                .increment()
                .accounts({ counter: counterPDA, authority: badActor.publicKey })
                .signers([badActor])
                .rpc()
            assert.fail('Should have thrown')
        } catch (err) {
            assert.include(err.message, 'ConstraintHasOne')
        }
    })
})
```

### 运行测试

```bash
# 启动本地节点并运行测试（最常用）
anchor test

# 不重启节点（复用已运行的 localnet）
anchor test --skip-local-validator

# 针对 devnet 测试
anchor test --provider.cluster devnet
```

---

## 2. BankClient（Rust 集成测试）

对于需要精确控制的场景，使用 Rust 写测试：

```rust
// tests/integration.rs
use solana_program_test::*;
use solana_sdk::{
    signature::Keypair,
    transaction::Transaction,
};

#[tokio::test]
async fn test_initialize() {
    let program_id = Pubkey::new_unique();
    let (mut banks_client, payer, recent_blockhash) =
        ProgramTest::new("my_program", program_id, processor!(process_instruction))
            .start()
            .await;

    let keypair = Keypair::new();
    let tx = Transaction::new_signed_with_payer(
        &[initialize_instruction(program_id, keypair.pubkey())],
        Some(&payer.pubkey()),
        &[&payer, &keypair],
        recent_blockhash,
    );

    banks_client.process_transaction(tx).await.unwrap();

    // 读取账户
    let account = banks_client.get_account(keypair.pubkey()).await.unwrap().unwrap();
    let state: MyState = MyState::try_from_slice(&account.data).unwrap();
    assert_eq!(state.count, 0);
}
```

---

## 3. 本地开发环境

```bash
# 启动本地测试网（Anchor 内置，或单独运行）
solana-test-validator

# 配置 CLI 使用 localnet
solana config set --url localhost

# 给测试账户充值
solana airdrop 10

# 部署程序到 localnet
anchor deploy
# 或
solana program deploy target/deploy/my_program.so

# 查看程序日志
solana logs
```

---

## 4. Devnet 部署

```bash
# 切换到 devnet
solana config set --url devnet

# 获取测试 SOL
solana airdrop 2

# 检查余额
solana balance

# 构建程序（优化版本）
anchor build

# 部署到 devnet
anchor deploy --provider.cluster devnet

# 同步 IDL（将 IDL 写入链上，供客户端下载）
anchor idl init --filepath target/idl/my_program.json $PROGRAM_ID \
  --provider.cluster devnet
```

---

## 5. 主网部署

```bash
# 切换到主网
solana config set --url mainnet-beta

# 检查余额（部署约需 2-4 SOL）
solana balance

# 构建
anchor build

# 生成新的程序密钥对（或使用已有的）
solana-keygen new -o target/deploy/my_program-keypair.json

# 主网部署
solana program deploy target/deploy/my_program.so \
  --keypair ~/.config/solana/id.json \
  --program-id target/deploy/my_program-keypair.json
```

---

## 6. 程序升级

Solana 程序默认是**可升级的**（upgrade authority 可以更新字节码）：

```bash
# 升级程序
solana program upgrade target/deploy/my_program.so $PROGRAM_ID

# 查看程序信息
solana program show $PROGRAM_ID

# 关闭可升级性（不可逆！将 upgrade authority 设为 None）
solana program set-upgrade-authority $PROGRAM_ID --final
```

**重要**：主网程序建议先放弃升级权，或将升级权转给多签（如 Squads Protocol），避免单点风险。

---

## 参考资源

- [Anchor 测试文档](https://www.anchor-lang.com/docs/testing)
- [Solana Program Test](https://docs.rs/solana-program-test/)
- [Squads Protocol（多签升级权）](https://squads.so/)
