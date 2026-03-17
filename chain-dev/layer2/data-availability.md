# 数据可用性

> **标签：** 数据可用性、da、eip-4844、celestia、eigenda、avail

**数据可用性（DA）** 保证「区块/批次中的交易数据可被网络中任何人获取与验证」。对 Rollup 而言，DA 是安全模型的核心组成部分。

---

## 1. 为什么 DA 很重要

**场景**：ZK Rollup 将有效性证明提交到 L1，但批次的原始交易数据没有发布。

- 任何人都无法独立重建 L2 状态
- Sequencer 可以作恶（隐藏数据），让用户无法证明自己的余额
- 这违背了 Rollup 继承 L1 安全性的核心承诺

**DA 的保证**：数据已发布 → 任何人在挑战期内可下载并验证。

---

## 2. DA 方案对比

| 方案 | 存储位置 | 可用性保证 | 成本 | 信任假设 |
|------|---------|-----------|------|---------|
| **L1 calldata** | 以太坊永久存储 | 最强（共识层） | 最贵 | 无额外信任 |
| **EIP-4844 blob** | 以太坊（~18 天后可裁剪）| 强 | 便宜（独立费用市场） | 无额外信任 |
| **Celestia** | 独立 DA 链 | DAS 采样验证 | 更便宜 | 信任 Celestia 验证者 |
| **EigenDA** | EigenLayer 再质押节点 | 节点承诺 + 经济惩罚 | 便宜 | 信任 EigenDA 运营者 |
| **Avail** | 独立 DA 链 | KZG + DAS | 便宜 | 信任 Avail 验证者 |
| **Validium（DAC）** | 链下委员会节点 | 委员会签名 | 最便宜 | N-of-M 委员会诚实 |

---

## 3. EIP-4844（Proto-Danksharding）

**Dencun 升级（2024 年 3 月）**正式上线：

### Blob 交易

```
EIP-4844 Blob 交易（Type 3）：
├── 普通 EVM calldata（很小，只存哈希）
├── blob 字段（最多 6 个 blob，每个约 128 KB）
└── KZG 承诺（证明 blob 数据正确）
```

### 关键参数

| 参数 | 值 |
|------|-----|
| 每个 blob 大小 | 128 KB（4096 个 field elements） |
| 目标 blob 数/块 | 3（初始），计划提高 |
| 最大 blob 数/块 | 6（初始） |
| blob 保留时间 | ~18 天（4096 个 epoch） |
| blob 独立费用 | `blob_base_fee` 独立调整（EIP-1559 类似机制） |

### 对 L2 成本的影响

EIP-4844 上线后，主流 L2 成本下降 10–100 倍：
- Arbitrum：平均 gas 从 ~$0.1 降至 ~$0.001
- OP Mainnet：类似降幅
- zkSync Era：状态差值压缩 + blob = 更低成本

---

## 4. Celestia

Celestia 是首个专用 DA 层，核心技术：

### 数据可用性采样（DAS）

```
1. 数据按纠删码（Reed-Solomon）编码，将 1MB 数据扩展为 2MB
2. 每个轻节点随机下载少量样本（如 64 个 512-byte chunk）
3. 若所有样本都能下载，以高概率证明 2MB 全量数据可用
4. 即使 50%+ 数据被隐藏，采样失败概率极高，诚实节点拒绝该区块
```

**优点**：轻节点可参与 DA 验证，无需下载全部数据。

### Celestia 架构

```
Rollup（如 Eclipse）
  ↓ 发布 blob（namespace + data）
Celestia 数据层
  ├── Celestia 验证者（共识）
  └── DAS 轻节点
以太坊（结算层）
  ↓ 验证 ZK 证明
```

---

## 5. EigenDA

EigenDA 基于 **EigenLayer** 再质押机制：

- ETH 质押者选择在 EigenDA 运营者处再质押（restaking）
- EigenDA 运营者承诺存储数据，违约则被惩罚
- **信任模型**：依赖 EigenLayer 经济安全，而非独立验证者网络

```
Rollup → EigenDA 分散者（Disperser） → 分发给多个运营者节点
                                         ↓
                              各运营者返回可用性签名（BLS）
                                         ↓
                         聚合签名写入以太坊（L1 验证）
```

---

## 6. 选型建议

| 优先级 | 方案 |
|--------|------|
| 最高安全（L1 安全等级） | EIP-4844 blob |
| 安全性强 + 低成本 | Celestia |
| EigenLayer 再质押生态 | EigenDA |
| 极低成本（接受 DAC 信任） | Validium（DAC） |

---

## 参考资源

- [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)
- [Celestia 文档](https://docs.celestia.org/)
- [EigenDA 文档](https://docs.eigenlayer.xyz/eigenda/overview)
- [Danksharding 路线图](https://ethereum.org/en/roadmap/danksharding/)
