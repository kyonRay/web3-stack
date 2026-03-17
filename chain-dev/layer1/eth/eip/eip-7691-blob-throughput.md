# EIP-7691：Blob 吞吐量提升

> **标签：** eip-7691、blob、数据可用性、L2、Pectra、吞吐量
> **所属升级：** Pectra（Prague-Electra，主网激活：2025.5.7）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 EIP-4844 的 blob 容量保守设计

EIP-4844（Dencun，2024.3）以 **3 target / 6 max blobs per block** 的参数上线，这是一个故意保守的选择：

```
EIP-4844 设计选择的原因：
  1. 安全边际：第一次引入 blob 机制，不确定节点实际带宽承受能力
  2. 共识风险控制：保守参数降低分叉失败风险
  3. 监控期：需要观察网络在真实 blob 负载下的表现
  4. KZG 证明计算：节点需要为所有 blob 计算 KZG 证明，CPU 开销需要评估

当时设计预期：先以小 blob 数量验证机制，后续视情况逐步扩容
```

### 1.2 L2 增长超出预期

Dencun 激活后，blob 使用量快速增长，原始容量的限制逐渐显现：

```
Dencun 激活后 blob 使用统计（2024 年数据）：
  平均 blob 使用率：约 80-90%（经常打满 6 blobs/block 上限）
  高峰期：连续多个 epoch 保持在 target 以上，base fee 飙升

主要 L2 每日 blob 需求：
  Arbitrum One：约 2-3 blobs/block
  OP Mainnet + Base：约 1-2 blobs/block
  zkSync Era：约 0.5-1 blob/block
  合计：远超 3 blobs 目标，常触碰 6 max 上限

结果：blob fee 周期性飙升，L2 费用随之上涨
```

L2 生态的爆发性增长（Base 月活用户突破 1000 万，Arbitrum 生态扩展等）使得 3 blobs 的目标显然不够，需要快速扩容。

### 1.3 为什么不直接等 PeerDAS

完整的数据可用性采样（PeerDAS，EIP-7594）计划在 Fusaka（2025.12）实现，届时可以大幅扩容。但：

```
2024 年中 → 2025.5 Pectra 激活：约 12 个月的等待期
在此期间，blob 容量不够用的问题每天都在发生

EIP-7691 的定位：Pectra 中的"快速扩容补丁"，不等 PeerDAS 先提升参数
网络监控数据表明：现有节点完全有能力处理 6 target / 9 max blobs，带宽有余量
```

---

## 2. 核心变更

### 2.1 blob 参数调整

| 参数 | Dencun（EIP-4844） | Pectra（EIP-7691） | 变化 |
|------|-------------------|------------------|------|
| `TARGET_BLOB_GAS_PER_BLOCK` | `393216`（3 blobs） | `786432`（6 blobs） | **+100%** |
| `MAX_BLOB_GAS_PER_BLOCK` | `786432`（6 blobs） | `1179648`（9 blobs） | **+50%** |
| 单 blob 大小 | 131,072 bytes（128 KB） | 131,072 bytes（128 KB） | 不变 |
| 每块 blob 目标带宽 | ~384 KB/block | ~768 KB/block | **+100%** |
| 每块 blob 最大带宽 | ~768 KB/block | ~1152 KB/block | **+50%** |

### 2.2 blob fee 调整参数更新

EIP-7691 同时调整了 blob fee market 的更新参数，以匹配新的 target/max 比例：

```
blob base fee 更新公式（EIP-1559 风格）：
  旧：调整幅度 = excess_blob_gas / TARGET_BLOB_GAS_PER_BLOCK / 64
  新：调整参数重新校准，保持与 EIP-1559 相同的价格稳定性特征

关键考虑：
  target 翻倍后，如果不调整更新参数，blob fee 会比预期更平滑（响应变慢）
  EIP-7691 确保价格发现机制在新参数下与 EIP-4844 激活时行为一致
```

### 2.3 实际日吞吐量变化

```
每块时间：12 秒
每天块数：86400 / 12 = 7200 块

EIP-4844（Dencun）：
  每日目标 blob 数：3 × 7200 = 21,600 blobs
  每日目标数据量：21,600 × 128 KB = 2.76 GiB

EIP-7691（Pectra）：
  每日目标 blob 数：6 × 7200 = 43,200 blobs
  每日目标数据量：43,200 × 128 KB = 5.53 GiB

对比：DA 目标容量翻倍（+100%）
```

---

## 3. 生效前后对比

| 维度 | EIP-7691 之前（Dencun） | EIP-7691 之后（Pectra） |
|------|----------------------|----------------------|
| **blob target** | 3/block | **6/block** |
| **blob max** | 6/block | **9/block** |
| **日 DA 目标容量** | ~2.76 GiB | ~5.53 GiB |
| **可容纳的 L2 活跃度** | 约 3-5 个主要 L2 | 约 6-10 个主要 L2 |
| **blob base fee 稳定性** | 3 blobs 目标下稳定 | 6 blobs 目标下稳定 |
| **L2 费用** | 高峰期 blob 打满时 fee 飙升 | 缓冲空间翻倍，fee 波动降低 |

### L2 生态容量估算

```
假设每个主要 L2 平均需要 ~1 blob/block（繁忙时）：

Dencun target（3 blobs）：
  可稳定服务约 3 个主要 L2
  高峰期任何 L2 增长都可能触及上限

Pectra target（6 blobs）：
  可稳定服务约 6 个主要 L2
  为未来 Optimism Superchain、Arbitrum Orbit 等生态扩展预留了空间
```

---

## 4. 影响范围

### 对 L2 开发者（最直接受益）

- **无需修改任何代码**：blob 参数变化对 L2 透明，直接生效
- **扩容空间**：可以增加每批次提交的数据量，或降低批次间隔，提升 L2 TPS
- **费用下降**：target 翻倍后，blob base fee 在相同负载下更低

### 对节点运营者

- **带宽要求提升**：平均每块需要处理的 blob 数据量翻倍
  - 目标：768 KB/block（vs. 384 KB）
  - 最大：1152 KB/block（vs. 768 KB）
- **存储需求轻微增加**：~18 天滚动窗口，存储量翻倍（约 100-200 GB → 200-400 GB）
- **CPU 开销**：KZG 证明计算增加，但现代硬件完全可以承受

### 对 DApp 开发者

- 使用 blob 发布数据的合约/服务直接受益（更多空间，更低费用）
- 无需代码修改

### 对用户

- L2 用户将看到手续费进一步降低（或在相同费用下获得更快的 L2 确认）
- 以太坊主网用户几乎感知不到差别（blob 是独立 fee market）

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-4844** | 直接扩展 | 7691 是对 4844 参数的更新，无法脱离 4844 独立存在 |
| **EIP-7623** | 配套政策 | 同在 Pectra 中，7623 提高 calldata 成本，迫使 L2 切换至 blob；7691 确保切换后有足够的 blob 空间 |
| **EIP-7594（PeerDAS）** | 后续演进 | Fusaka 的 PeerDAS 通过数据可用性采样实现进一步 blob 扩容（更大幅度） |
| **EIP-7892（BPO）** | 长期规划 | BPO 分叉机制允许无需硬分叉调整 blob 参数，EIP-7691 是在 BPO 成熟前的过渡 |
| **EIP-1559** | 设计参考 | blob fee market 机制与 EIP-1559 相同，7691 的参数调整也遵循相同原则 |

### Pectra blob 扩容组合拳

```
EIP-7691：target 3→6，max 6→9（供给侧）
EIP-7623：calldata 成本大幅上涨（需求侧引导）

两者协同效果：
  1. 更多 L2 必须转向 blob（calldata 太贵）
  2. 更多 blob 空间可以容纳这些 L2（7691 扩容）
  3. 整体 DA 成本下降，L2 手续费进一步降低
```

---

## 参考资源

- [EIP-7691 原文](https://eips.ethereum.org/EIPS/eip-7691)
- [EIP-4844 blob 使用追踪](https://blobscan.com/)
- [Blob 费用监控](https://ultrasound.money/)
- [L2 DA 成本追踪](https://l2fees.info/)
