# EIP-1559：费用市场改革

> **标签：** eip-1559、gas、base-fee、燃烧、London、费用机制
> **所属升级：** London（主网激活：2021.8.5，区块 12965000）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 旧模型：First-Price Auction（第一价格拍卖）

EIP-1559 之前，以太坊使用**单一 gas price 竞价模型**：用户设定一个 gas price，矿工优先打包出价最高的交易。这个看似简单的机制存在三个根本性缺陷：

**缺陷 1：费用不可预测**

```
用户发交易时估算：当前 gas price 约 30 Gwei
提交后网络突然拥堵：gas price 飙升至 150 Gwei
结果：交易长时间悬挂，或被迫取消重发（FOMO 重价）
```

钱包软件需要复杂的预测算法估算"合适"的 gas price，但在快速变化的市场中经常失败，导致用户交易要么超时要么超支。

**缺陷 2：系统性超额支付**

第一价格拍卖在机制设计上是低效的：用户支付的是自己出价，而非市场出清价格。为了保证交易被打包，用户被迫出价远高于实际需要的价格（通常高出 2-3 倍作为"保险金"）。

**缺陷 3：矿工操纵风险**

矿工可以通过**人为填充区块**（打包自己的零价值交易）来维持高 gas price，或与大型用户私下谈判，形成寻租空间。由于矿工获得全部交易费，这种操纵存在直接的经济激励。

**缺陷 4：区块大小固定导致人为稀缺**

旧模型的区块 gas limit 是固定上限，没有目标值概念。当需求超过上限时，只能通过价格竞争排队，无法通过弹性区块大小来平滑短期波动。

---

## 2. 核心变更

### 2.1 双层费用结构

EIP-1559 将单一 gas price 拆分为两个独立组件：

```
交易费 = (base fee + priority fee) × gas used

base fee（基础费）：
  - 协议自动计算，所有用户相同
  - 全额销毁（不进入任何账户）
  - 随区块利用率自动调整

priority fee（优先费 / 小费）：
  - 用户自主设定
  - 支付给验证者（区块提案者）
  - 通常 1-2 Gwei 即可，紧急交易可设高
```

用户实际设置两个参数：

| 参数 | 说明 | 典型值 |
|------|------|--------|
| `maxFeePerGas` | 愿意支付的每 gas 上限 | base fee × 2 + tip |
| `maxPriorityFeePerGas` | 给验证者的小费 | 1-2 Gwei |

实际支付：`min(maxFeePerGas, base fee + maxPriorityFeePerGas) × gas used`

用户设置 `maxFeePerGas` 作为"保护上限"，超过这个价格时交易等待，不会意外超支。

### 2.2 base fee 算法

base fee 按如下规则每区块自动调整：

```
target gas per block = 15,000,000（区块 gas limit 的一半）
max gas per block    = 30,000,000（弹性上限）

if parent_gas_used > target:
    base_fee = parent_base_fee × (1 + (parent_gas_used - target) / target / 8)
else:
    base_fee = parent_base_fee × (1 - (target - parent_gas_used) / target / 8)

最大单块变化幅度：±12.5%
```

当区块满载（30M gas）时，base fee 每块最多上涨 12.5%；当区块为空时，每块最多下降 12.5%。这种机制使 base fee 追踪真实需求，而非被人为操纵。

### 2.3 弹性区块大小

| 指标 | 旧模型 | EIP-1559 |
|------|--------|---------|
| 区块 gas 目标 | 无（固定上限） | 15M（目标） |
| 区块 gas 上限 | 15M（固定） | 30M（2x 弹性） |

弹性大小允许区块在高峰期短暂扩展，将短期需求波动吸收为区块大小变化而非价格飞涨，显著改善用户体验。

### 2.4 对合约开发的影响

```solidity
// EIP-1559 前：tx.gasprice = 用户设定的单一 gas price
uint256 gasCost = tx.gasprice * gasUsed;  // 矿工全部获得

// EIP-1559 后：tx.gasprice = base fee + priority fee（实际支付的有效 gas price）
// base fee 已被销毁，合约中 tx.gasprice 语义不变，但矿工只得到 priority fee 部分
// 新增：block.basefee 可查询当前区块的 base fee
uint256 currentBaseFee = block.basefee;  // Solidity 0.8.7+ 可用
```

---

## 3. 生效前后对比

| 维度 | EIP-1559 之前 | EIP-1559 之后 |
|------|--------------|--------------|
| **费用结构** | 单一 gas price | base fee（销毁）+ priority fee（给验证者） |
| **定价机制** | 用户竞价，矿工选价高者 | 协议自动定价 base fee，用户仅设小费 |
| **费用可预测性** | 差（块间可波动 10x+） | 较好（块间变化幅度上限 ±12.5%） |
| **超额支付** | 系统性（2-3x 保险出价） | 最多多付 1 个 base fee 的差额 |
| **区块大小** | 固定上限（15M gas） | 弹性（目标 15M，上限 30M gas） |
| **矿工/验证者收入** | 全部交易费 | 仅 priority fee（小费） |
| **ETH 通缩压力** | 无 | base fee 全额销毁（高峰期 ETH 净通缩） |
| **矿工操纵激励** | 高（操纵 gas price 有直接收益） | 低（操纵 base fee 需改变区块利用率，base fee 不入账） |
| **交易等待时间** | 低 gas price 可能等数小时 | `maxFeePerGas` > base fee 即可快速打包 |
| **钱包估算难度** | 高（复杂预测算法） | 低（只需预测下一块 base fee，误差小） |

### 实际 gas price 示例

```
场景：当前 base fee = 20 Gwei，用户设置：
  maxFeePerGas         = 50 Gwei
  maxPriorityFeePerGas = 2 Gwei

实际有效 gas price = min(50, 20 + 2) = 22 Gwei
  → 矿工获得：2 Gwei × gas used
  → 销毁：20 Gwei × gas used
  → 用户返还：(50 - 22) Gwei × gas used（未使用的 maxFee 退还）
```

---

## 4. 影响范围

### 对链开发者

- **客户端实现**：需实现 base fee 计算逻辑、弹性区块 gas limit、销毁机制
- **区块结构**：区块头新增 `baseFeePerGas` 字段
- **交易类型**：引入 Type-2 交易（EIP-2930 Type-1 基础上新增 `maxFeePerGas` / `maxPriorityFeePerGas`）

### 对合约开发者

- `tx.gasprice`：语义保持不变（实际有效 gas price），但含义已改变（不再是用户出价，而是 `base fee + tip`）
- `block.basefee`：新增全局变量，可查询当前区块 base fee（Solidity 0.8.7+）
- 合约内 gas 计算逻辑一般不受影响，但依赖 `tx.gasprice` 做 gas 激励的合约需要重新审计

### 对 DApp 开发者

```typescript
// ethers.js v6 / viem 自动处理 EIP-1559 费用估算
const feeData = await provider.getFeeData()
// feeData.maxFeePerGas = 预估 base fee × 2 + maxPriorityFeePerGas
// feeData.maxPriorityFeePerGas = 建议小费

// viem 示例
const gasPrice = await publicClient.estimateFeesPerGas()
// { maxFeePerGas, maxPriorityFeePerGas }
```

### 对用户

- **正面**：交易费预测更容易，超支风险降低，等待时间更稳定
- **负面**：区块 gas limit 翻倍（15M→30M）在短期内增加了节点同步压力

### 对验证者/矿工（过渡期）

- 收入结构改变：全部交易费 → 仅 priority fee（小费）
- 在 ETH 价格/网络活跃度不变的情况下，短期内矿工/验证者收入下降

### 对 ETH 经济模型

- 高负载时期（如 NFT 热潮、DeFi 峰值）ETH 实现净销毁，转变为通缩资产
- "超声波货币"（Ultrasound Money）叙事的技术基础

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-2930** | 前置（Type-1 交易） | EIP-1559 的 Type-2 交易基于 EIP-2930 的 access list 扩展 |
| **EIP-4844** | 延伸应用 | EIP-4844 的 blob fee market 复用了相同的 EIP-1559 风格自动定价机制 |
| **EIP-7691** | 参数调整 | EIP-7691 调整了 blob fee 更新参数，延续 EIP-1559 设计思路 |
| **EIP-1559 → PoS 验证者** | 重要性变化 | The Merge 后矿工改为验证者，EIP-1559 使验证者收入更稳定（基础收益来自 PoS 奖励，而非交易费） |

---

## 参考资源

- [EIP-1559 原文](https://eips.ethereum.org/EIPS/eip-1559)
- [Vitalik：EIP-1559 FAQ](https://notes.ethereum.org/@vbuterin/eip-1559-faq)
- [Tim Roughgarden：EIP-1559 经济分析论文](https://timroughgarden.org/papers/eip1559.pdf)
- [Etherscan Gas Tracker](https://etherscan.io/gastracker)
