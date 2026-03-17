# EIP-4844：Proto-Danksharding（Blob 交易）

> **标签：** eip-4844、blob、proto-danksharding、KZG、数据可用性、L2、Dencun
> **所属升级：** Dencun（主网激活：2024.3.13，slot 8626176）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 L2 的数据发布瓶颈

以太坊的 Rollup 扩容路线（Rollup-Centric Roadmap）依赖 L2 将批量交易数据发布到 L1，供任何人验证。EIP-4844 之前，L2 只能将数据以 **calldata** 的形式写入以太坊交易，而 calldata 存在两个根本问题：

**问题 1：calldata 永久存储，成本极高**

```
L1 calldata 写入成本（EIP-1559 之后）：
  非零字节：16 gas/byte
  零字节：  4 gas/byte

一个 Arbitrum 批次约 100KB：
  100,000 bytes × 平均 ~12 gas/byte = 1,200,000 gas
  × 50 Gwei base fee × 约 $3,000/ETH ≈ $180 per batch

每秒处理 ~2000 笔 L2 交易 → 每笔均摊约 $0.09 的 L1 DA 成本
高峰期（base fee 200 Gwei+）单笔 L2 费用超 $2
```

所有节点必须**永久下载和存储**这些 calldata，即使它只是 L2 的临时证明数据——对以太坊的长期存储压力极大。

**问题 2：calldata 与 EVM 执行混用，无法独立定价**

calldata 费用受主网 EVM 执行的 gas 竞争影响：DeFi 活跃时 gas price 飙升，L2 DA 成本随之暴涨，两者的需求耦合导致 L2 用户体验直接受 L1 活动影响。

### 1.2 为什么不直接降低 calldata 成本

降低 calldata 成本会增加最坏情况区块大小，加重节点同步压力，与以太坊的去中心化目标冲突。需要的是**一种专为临时数据设计的存储方式**，它足够廉价，且在 L2 完成验证后可以丢弃。

---

## 2. 核心变更

### 2.1 新交易类型：Type-3（Blob 携带交易）

EIP-4844 引入了第三种交易类型（`0x03`），在普通交易基础上附加 **blob sidecar**：

```
Type-3 交易结构
├── 普通交易字段（nonce, gas, to, value, data...）
├── max_fee_per_blob_gas（blob gas 上限）
├── blob_versioned_hashes[]（blob 内容的 KZG 承诺哈希列表）
└── blob sidecar（链下附带）
    ├── blobs[]（实际数据，不进入 EVM）
    ├── commitments[]（KZG 承诺）
    └── proofs[]（KZG 证明）
```

关键设计：**blob 数据本身不进入 EVM 执行层**，合约只能访问 `blobhash(index)` 这个 32 字节的承诺哈希，无法直接读取 blob 内容。

### 2.2 Blob 技术规格

| 参数 | 值 | 说明 |
|------|-----|------|
| 每个 blob 大小 | 131,072 bytes（128 KB） | 由 4096 个 32 字节 field elements 组成 |
| 每块 blob 目标 | 3（Dencun 激活时） | EIP-7691 后升为 6 |
| 每块 blob 上限 | 6（Dencun 激活时） | EIP-7691 后升为 9 |
| blob 保留时长 | ~18 天（4096 个 epoch） | 足够 L2 完成欺诈证明/有效性证明 |
| 存储位置 | 共识层（信标节点） | 不在执行层永久存储 |
| 验证方式 | KZG 多项式承诺 | 48 字节承诺可高效验证 128KB 数据 |

### 2.3 独立 Blob Fee Market

Blob 有独立的 gas 市场，与普通 EVM gas 完全隔离：

```
blob_base_fee 更新规则（类 EIP-1559）：

if parent_blob_gas_used > target:
    blob_base_fee *= (1 + excess / target / 64)  # 缓慢上涨
else:
    blob_base_fee *= (1 - gap / target / 64)     # 缓慢下降

MIN_BLOB_BASE_FEE = 1 wei（极低起点）
```

blob fee market 独立调整，DeFi 热潮时 EVM gas 飙升不会直接带动 blob 费用上升。

### 2.4 KZG 承诺机制

KZG（Kate-Zaverucha-Goldberg）多项式承诺是 EIP-4844 的密码学核心：

```
一个 blob = 一个次数 ≤ 4095 的多项式 p(x)
  每个 field element 是多项式在某个点的值

KZG 承诺 = g^p(τ)（在可信设置的 τ 点求值）
  大小：48 字节（一个 BLS12-381 G1 点）
  可以在不知道 blob 全部内容的情况下验证特定点的值

KZG 证明验证（链上，EIP-4844 新增预编译）：
  输入：commitment, point, value, proof
  验证：commitment 确实在 point 处等于 value
  Gas 成本：约 50,000 gas（远低于 Solidity 实现）
```

### 2.5 合约中访问 blob

```solidity
// blob 哈希就是对应 blob 的 KZG 承诺的版本化哈希
// 格式：0x01 || keccak256(commitment)[1:]
bytes32 blobHash0 = blobhash(0); // 第 0 个 blob 的版本化哈希
bytes32 blobHash1 = blobhash(1); // 第 1 个 blob

// L2 合约验证示例：确认某个批次数据对应某个 blob
// （合约只知道哈希，不知道内容）
require(blobhash(0) == expectedBlobHash, "Wrong blob");
```

---

## 3. 生效前后对比

| 维度 | EIP-4844 之前 | EIP-4844 之后 |
|------|--------------|--------------|
| **L2 数据发布方式** | calldata（永久存储于执行层） | blob（临时存储于共识层，~18 天） |
| **DA 成本** | ~16 gas/非零字节（$0.50-$2.00/L2 swap 高峰期） | ~$0.001-$0.05/L2 swap（降低 10-100x） |
| **数据存储期限** | 永久（每个节点都要存储） | ~18 天（节点可裁剪） |
| **L1/L2 费用耦合** | 强耦合（L1 拥堵直接推高 L2 费用） | 解耦（独立 blob fee market） |
| **区块 DA 容量** | 无专用空间（受 calldata limit 约束） | 3 target / 6 max blobs = ~384 KB 专用空间（Dencun 时） |
| **EVM 内访问** | calldata 可在 EVM 内读取 | blob 内容不可在 EVM 读取（只能获取哈希） |
| **验证方式** | 重新执行即可验证 | KZG 承诺证明（密码学保证） |
| **节点存储压力** | 随时间线性增长（永久） | 稳定（18 天滚动窗口） |

### 实际费用对比（Arbitrum One 示例）

| 时间点 | 简单 ETH 转账 | Swap |
|--------|-------------|------|
| Dencun 之前（2024.2，高峰） | ~$0.15 | ~$1.50 |
| Dencun 之后（2024.3） | ~$0.002 | ~$0.02 |
| 降幅 | ~75x | ~75x |

---

## 4. 影响范围

### 对 L2 开发者（最直接影响）

```python
# L2 Rollup 数据发布策略变化

# 之前：用 calldata 发布批次
tx = {
    "to": L1_INBOX,
    "data": batch_data,  # 直接放 calldata，永久存储
    "gasPrice": high_gas_price,
}

# 之后：用 blob 发布批次
tx = {
    "to": L1_INBOX,
    "data": b"",  # 可以为空或仅存哈希
    "blobs": [batch_data_blob],   # 实际数据作为 blob
    "maxFeePerBlobGas": blob_fee,
}
# OP Stack、Arbitrum、zkSync 均已在 Dencun 后切换 blob 模式
```

### 对执行客户端开发者

- 需实现 Type-3 交易解析与广播
- blob 的 KZG 证明验证（新增预编译 `0x0a`）
- blob gas 的独立计费与区块限制检查

### 对共识客户端开发者

- 信标节点需存储 blob sidecar，并在 ~18 天后裁剪
- `getBlobSidecarsByBlockRoot` 等新 API

### 对合约开发者

- 新操作码 `BLOBHASH`（0x49）可在合约中访问 blob 哈希
- 新预编译 `0x0a`：`point_evaluation`，用于验证 KZG 证明
- 无需修改现有合约，但可利用 blob 设计新的 DA 验证合约

### 对 DApp 开发者

```typescript
// viem 发送 blob 交易
import { parseGwei, stringToHex } from 'viem'
import { toBlobs, blobsToCommitments, commitmentsToVersionedHashes } from 'viem'

const blobs = toBlobs({ data: batchData })

const hash = await walletClient.sendTransaction({
  type: 'eip4844',
  to: L1_INBOX,
  blobs,
  kzg,  // KZG trusted setup
  maxFeePerBlobGas: parseGwei('0.001'),
})
```

### 对最终用户

- L2 手续费大幅降低（10-100x）
- 感知不到 blob 机制本身，只感受到 L2 费用下降

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-1559** | 设计复用 | blob fee market 使用相同的自动调整机制 |
| **EIP-7691** | 直接扩展 | Pectra 将 blob target 3→6，max 6→9（EIP-4844 的参数升级） |
| **EIP-7623** | 配套政策 | 提高 calldata 成本，强制 L2 切换 blob |
| **EIP-7594（PeerDAS）** | 后续演进 | Fusaka 引入点对点采样，进一步扩展 blob 容量 |
| **Full Danksharding** | 终极目标 | 64+ blobs/block + 数据可用性采样，百倍以上扩容 |
| **EIP-4337** | 无直接关联 | 账户抽象与 blob 并行演进，互不干扰 |

### Proto-Danksharding vs Full Danksharding

```
Proto-Danksharding（EIP-4844，已上线）
├── KZG 承诺基础设施 ✓
├── 独立 blob fee market ✓
└── 每个节点下载所有 blob（带宽瓶颈）

Full Danksharding（未来）
├── 数据可用性采样（DAS）→ 每个节点只需下载 1/N 的数据
├── 64+ blobs per block
└── 数百万 TPS 的 L2 理论上限

PeerDAS（EIP-7594，Fusaka 已上线）
└── DAS 的第一步，使用 P2P 列采样取代全量下载
```

---

## 参考资源

- [EIP-4844 原文](https://eips.ethereum.org/EIPS/eip-4844)
- [eip4844.com 可视化介绍](https://www.eip4844.com/)
- [KZG 可信设置仪式](https://ceremony.ethereum.org/)
- [Dankrad Feist：数据可用性采样详解](https://dankradfeist.de/ethereum/2020/10/13/files/das-requirements.pdf)
- [Rollup 数据成本追踪](https://l2fees.info/)
