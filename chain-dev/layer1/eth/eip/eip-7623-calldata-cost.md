# EIP-7623：提高 Calldata 成本

> **标签：** eip-7623、calldata、gas、区块大小、DA、Pectra
> **所属升级：** Pectra（Prague-Electra，主网激活：2025.5.7）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 Calldata 的双重角色冲突

Calldata 在以太坊中承担两种截然不同的角色：

**角色 1：EVM 执行参数**（本职工作）
```
transfer(address to, uint256 amount) 的 calldata：
  函数选择器：4 bytes
  to 参数：32 bytes
  amount 参数：32 bytes
  合计：68 bytes → 这是合约交互的必要开销
```

**角色 2：数据可用性（DA）载体**（历史扭曲）
```
L2 Rollup 早期的 calldata 批次：
  Arbitrum 一个批次：~50,000-200,000 bytes
  Optimism 一个批次：~20,000-100,000 bytes
  
  这些字节放入 calldata 是为了"写入以太坊作为 DA"
  → 与 EVM 执行完全无关，只是利用 calldata 的永久存储特性
```

这种双重角色导致了结构性问题：**大型 calldata 交易（DA 用途）可以将以太坊区块撑到极端大小**。

### 1.2 极端区块大小风险

```
EIP-4844 之前的最坏情况：
  calldata gas limit 上限：约 1.5M gas（非零字节 16 gas/byte）
  → 最大 calldata：约 93,750 bytes per tx
  
  如果整个区块都是 calldata 密集交易：
  区块 gas limit 15M → 约 937,500 bytes（~1 MB）calldata
  
  更极端的情况（低 gas price 时代）：
  攻击者可用极低成本填充极大区块，导致：
    × 区块传播延迟（P2P 广播需要更长时间）
    × 历史同步慢（新节点同步代价高）
    × 网络分叉风险
```

EIP-4844 后，L2 被鼓励切换到 blob，但**没有强制机制**。部分 L2 或数据发布服务仍然可能选择使用 calldata（在极端情况下比 blob 便宜），使极端区块问题依然存在。

### 1.3 Calldata 与 Blob 的价格竞争

```
EIP-4844 之后，在 blob 使用率低时：
  blob fee 极低（可以低至 1 wei/blob）
  calldata fee 仍保持原价（16 gas/非零字节）

  → blob 远比 calldata 便宜，L2 有动力切换

但在 blob 拥堵时（blob base fee 飙升）：
  blob fee 可能高于 calldata fee
  → L2 可能退回到 calldata 作为备选
  
这种"blob/calldata 套利切换"行为会导致区块大小极端不稳定
```

---

## 2. 核心变更

### 2.1 "仅含 calldata 交易"的地板价提升

EIP-7623 引入了针对**calldata 密集型交易**的特殊计费逻辑：

```
定义"tokens"数量：
  tokens = 非零字节数 × 4 + 零字节数 × 1

新的 calldata 地板价计算：
  floor_cost = tokens × STANDARD_TOKEN_COST

  其中 STANDARD_TOKEN_COST = 10 gas/token（等效于：非零字节 40 gas，零字节 10 gas）

规则：
  如果 tokens × STANDARD_TOKEN_COST > tx.gasLimit × EXECUTION_GAS_ALLOWANCE:
    该交易必须至少支付 floor_cost 的 gas（即使实际 EVM 执行 gas 更少）
```

**简化理解**：

| 字节类型 | EIP-7623 之前 | EIP-7623 之后（地板价） |
|---------|--------------|----------------------|
| 非零字节 | 16 gas/byte | **40 gas/byte**（+150%） |
| 零字节 | 4 gas/byte | **10 gas/byte**（+150%） |

注意：这是**地板价**（最低收费），不是所有交易都受影响——只有 calldata 比例极高的"数据密集型"交易才触发。

### 2.2 哪些交易受影响

```
受影响（触发地板价）：
  ✗ L2 Rollup 批次提交（大量 calldata，几乎无 EVM 执行）
  ✗ 数据发布服务（直接用 calldata 存储数据）
  ✗ 极端 calldata 填充攻击

不受影响（正常 EVM 执行交易）：
  ✓ 普通 ETH 转账（calldata 极少）
  ✓ 标准合约交互（approve、transfer、swap 等）
  ✓ NFT mint（calldata 少，执行多）
  ✓ 大多数 DApp 交互（执行 gas >> calldata gas）
```

### 2.3 区块大小上限收紧

EIP-7623 通过提高 calldata 地板价，间接降低了最坏情况下的区块大小：

```
EIP-7623 之前（最坏情况区块）：
  每个 L2 批次使用 calldata：~200 KB
  30M gas 区块 / 200KB 的 gas 成本 ≈ 多个巨型交易
  最坏情况：~1.79 MB calldata per block

EIP-7623 之后（最坏情况区块）：
  calldata 地板价提升 2.5x
  相同 gas 下，calldata 容量降低 2.5x
  估算最坏情况：约 0.6 MB calldata per block
```

---

## 3. 生效前后对比

| 维度 | EIP-7623 之前 | EIP-7623 之后 |
|------|--------------|--------------|
| **非零字节 calldata 地板价** | 16 gas/byte | **40 gas/byte** |
| **零字节 calldata 地板价** | 4 gas/byte | **10 gas/byte** |
| **DA 用途 calldata 成本** | 相对低廉（可作为 blob 的备选） | 显著高于同等 blob 成本 |
| **普通合约交互** | 正常（calldata 比例小） | **不受影响**（地板价不触发） |
| **L2 batch 成本** | 可以在 blob fee 高时切换到 calldata | blob 几乎始终更便宜 |
| **最坏情况区块大小** | ~1.79 MB calldata | ~0.6 MB calldata（降低 ~66%） |
| **区块传播稳定性** | 极端情况可能出现超大块 | 上限收紧，传播更可预测 |

### 对 L2 数据发布成本的影响

```
场景：L2 发布一个 200KB 批次

方式 A：calldata（EIP-7623 之后）
  非零字节（假设 60%）：120,000 bytes × 40 gas = 4,800,000 gas
  零字节（40%）：80,000 bytes × 10 gas = 800,000 gas
  合计：~5,600,000 gas（约 11% 的区块 gas limit）

方式 B：blob（EIP-4844 + EIP-7691）
  200KB ≈ 1.5 blobs（每 blob 128KB）
  blob gas 独立计费，与 EVM gas 分离
  blob base fee 极低时：约等于 0（几乎免费）

结论：calldata 对于大型 DA 用途已完全失去竞争力
```

---

## 4. 影响范围

### 对 L2 开发者

- **强制切换 blob**：calldata 对 DA 用途过于昂贵，不再是 blob 的可行备选
- **无缝影响**：现有 L2 已在 Dencun 后切换到 blob，EIP-7623 只是进一步强化这个方向
- **设计简化**：不再需要维护 calldata/blob 双路径的 DA 策略

### 对普通合约/DApp 开发者

- **几乎无影响**：标准合约交互的 calldata 量相对于执行 gas 很小，地板价不会触发
- **合约参数传递**：正常的函数调用参数（几百字节）完全不受影响

### 对合约开发者（特殊情况）

```solidity
// 高度数据密集型合约（如链上数据存储）需要注意
// 如果合约设计是接收大量 calldata 但执行 gas 很少：

// 之前：成本 = calldata bytes × 16 gas
// 之后：成本可能是 calldata bytes × 40 gas（地板价触发）

// 评估：如果你的合约像这样...
function storeData(bytes calldata data) external {
    // data 很大（如 10KB+），但 EVM 执行只需 ~5000 gas
    emit DataStored(data);  // 只发 event，不写 storage
}
// → 受 EIP-7623 影响，需要考虑用 blob 替代方案
```

### 对节点运营者

- **利好**：区块传播更稳定，最坏情况区块大小降低
- **历史同步**：未来新节点同步时，calldata 密集区块的频率降低

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-4844** | 配套引导 | 4844 提供了 blob 机制；7623 通过提高 calldata 价格强制 L2 切换 blob |
| **EIP-7691** | 配套扩容 | 7691 增加 blob 容量；7623 增加 calldata 需求转向 blob 的动力；两者协同 |
| **EIP-1559** | 基础机制 | calldata 地板价基于现有 gas 机制，与 EIP-1559 base fee 叠加 |
| **Full Danksharding** | 长期目标 | 通过 EIP-7623 将 DA 完全导向 blob 路线，是迈向 Full Danksharding 的重要一步 |

---

## 参考资源

- [EIP-7623 原文](https://eips.ethereum.org/EIPS/eip-7623)
- [EIP-7623 讨论帖（Ethereum Magicians）](https://ethereum-magicians.org/t/eip-7623-increase-calldata-cost/18647)
- [Blobscan - blob 使用统计](https://blobscan.com/)
- [Rollup 数据成本追踪](https://l2fees.info/)
