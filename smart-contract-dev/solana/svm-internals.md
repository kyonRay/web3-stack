# SVM 深入：Sealevel 并行执行

> **标签：** svm、sealevel、并行执行、计算单元、账户模型

Solana 的**Sealevel VM（SVM）**是其高性能的核心秘密，通过声明账户依赖实现大规模并行交易执行。

---

## 1. 并行执行原理

EVM（以太坊）顺序执行交易，Solana 并行执行：

```
EVM（顺序）：
  Tx1 → Tx2 → Tx3 → ...
  （每笔交易可能修改任意状态，无法并行）

Sealevel（并行）：
  Tx1 [A, B]    ← 可与 Tx3 并行（无共享账户）
  Tx2 [B, C]    ← 与 Tx1/Tx3 有冲突，需等待
  Tx3 [D, E]    ← 可与 Tx1 并行
```

**关键前提**：每笔交易必须**预先声明**它会读/写哪些账户。若两笔交易无共同账户（无冲突），可并行执行。

---

## 2. 账户模型

与 EVM 最大的区别：

| 维度 | EVM | Solana |
|------|-----|--------|
| 状态位置 | 合约内（`mapping`、变量） | 独立账户（Account） |
| 程序是否有状态 | 有状态 | 无状态（状态在 Account） |
| 账户大小 | 动态（存储可增长） | 固定（创建时声明大小，含 realloc） |
| 租金模型 | 无 | 需要租金豁免余额（基于数据大小） |

```
Account 结构：
├── lamports:    SOL 余额（用于支付租金和转账）
├── data:        字节数组（程序自定义存储）
├── owner:       哪个程序拥有该账户（可读写 data）
├── executable:  是否是程序账户
└── rent_epoch:  租金相关（现代版本通常豁免）
```

---

## 3. 计算单元（Compute Units）

类似 EVM 的 gas，Solana 用**计算单元（CU）**计量指令成本：

| 操作 | 成本（约） |
|------|----------|
| 指令处理开销 | 1,000 CU |
| 日志记录（每个字节） | ~1 CU |
| keccak256 哈希 | ~2,500 CU |
| Secp256k1 签名验证 | ~6,000 CU |
| CPI 调用 | ~1,000 CU（+ 被调用程序成本） |
| Token Program 转账 | ~3,000 CU |

**默认限制**：每条指令 200,000 CU，每笔交易 1,400,000 CU。

### 设置计算单元限制与优先费

```typescript
import { ComputeBudgetProgram } from '@solana/web3.js'

// 设置更高的 CU 上限（复杂交易）
const modifyCU = ComputeBudgetProgram.setComputeUnitLimit({ units: 400_000 })

// 设置优先费（高峰期确保入块）
const addPriorityFee = ComputeBudgetProgram.setComputeUnitPrice({
    microLamports: 100_000 // 每个 CU 的优先费（microLamports）
})

const tx = new Transaction().add(modifyCU, addPriorityFee, myInstruction)
```

---

## 4. 指令处理流程

```
1. 客户端构建交易（包含指令 + 账户列表 + 签名）
2. 提交到 Validator RPC
3. Validator 广播到 Leader（当前负责出块的节点）
4. Banking Stage 并行执行：
   a. 检查账户冲突，构建并行执行调度
   b. 用 Sealevel 并行执行无冲突交易
   c. 顺序执行有冲突的交易
5. 生成区块，广播给其他 Validator
6. 通过 Turbine 协议传播区块
```

---

## 5. PoH（历史证明）

Solana 独有的 **Proof of History** 是一个 VDF（可验证延迟函数），为所有事件提供可验证的时间戳：

```
sha256(state[0]) = state[1]
sha256(state[1]) = state[2]
...
sha256(state[n]) = state[n+1]

每个 state 都嵌入了对应时间的事件（交易），使节点无需通信即可验证顺序
```

PoH 并不是共识机制（共识是 Tower BFT），而是**时钟**，让整个网络对时间顺序达成共识，从而加速 Validator 验证速度。

---

## 6. 开发注意事项

**账户大小**：
```rust
// 创建时必须指定大小，后续不可随意扩展
// Anchor 提供 realloc 功能（有限度扩展）
#[account(
    mut,
    realloc = 8 + new_size,
    realloc::payer = payer,
    realloc::zero = false,
)]
pub account: Account<'info, MyAccount>,
```

**Stack 大小限制**：
- Solana 程序栈大小为 4KB，大型结构体需用 `Box<T>` 堆分配
- 复杂 Anchor 结构可能触发栈溢出，需要 `Box<Account<'info, T>>`

**CPI 深度限制**：最大 CPI 调用深度为 4 层。

---

## 参考资源

- [Solana 架构文档](https://docs.solana.com/cluster/overview)
- [Sealevel 并行执行论文](https://medium.com/solana-labs/sealevel-parallel-processing-thousands-of-smart-contracts-d814b378192)
- [PoH 说明](https://solana.com/news/proof-of-history)
