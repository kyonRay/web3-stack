# 以太坊 EIP 详解

> 本目录收录以太坊核心 EIP 的深度解析，每篇文档涵盖问题背景、实现原理、前后对比与影响范围。

---

## 文档索引

### London 升级（2021.8.5）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-1559-fee-market.md](./eip-1559-fee-market.md) | EIP-1559 | 费用市场改革：base fee + 燃烧机制 |

### Shanghai/Capella 升级（2023.4.12）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-4895-beacon-withdrawals.md](./eip-4895-beacon-withdrawals.md) | EIP-4895 | 信标链质押提款解锁 |

### Dencun 升级（2024.3.13）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-4844-proto-danksharding.md](./eip-4844-proto-danksharding.md) | EIP-4844 | Proto-Danksharding：Blob 交易，L2 DA 成本降低 10-100x |

### 账户抽象（跨升级）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-4337-account-abstraction.md](./eip-4337-account-abstraction.md) | EIP-4337 | 账户抽象（合约级，无协议修改） |
| [eip-7702-eoa-code.md](./eip-7702-eoa-code.md) | EIP-7702 | EOA 账户代码委托（Pectra 协议级） |

### Pectra 升级（2025.5.7）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-7702-eoa-code.md](./eip-7702-eoa-code.md) | EIP-7702 | EOA 临时委托合约代码执行 |
| [eip-7251-max-effective-balance.md](./eip-7251-max-effective-balance.md) | EIP-7251 | 验证者最大有效余额 32→2048 ETH |
| [eip-7691-blob-throughput.md](./eip-7691-blob-throughput.md) | EIP-7691 | Blob 吞吐量翻倍（target 3→6，max 6→9） |
| [eip-7623-calldata-cost.md](./eip-7623-calldata-cost.md) | EIP-7623 | Calldata 地板价提升，驱动 L2 切换 blob |
| [eip-2537-bls-precompile.md](./eip-2537-bls-precompile.md) | EIP-2537 | BLS12-381 曲线预编译，支持 ZK 友好签名 |

### Fusaka 升级（2025.12.3）

| 文档 | EIP | 说明 |
|------|-----|------|
| [eip-7594-peerdas.md](./eip-7594-peerdas.md) | EIP-7594 | PeerDAS：点对点数据可用性采样，blob 容量理论 8x |

---

## 以太坊主网升级时间线

```
2015.7   Genesis（创世块）
2016.3   Homestead
2016.10  Tangerine Whistle / Spurious Dragon（Gas 重定价）
2019.2   Constantinople / St. Petersburg
2019.12  Istanbul（EIP-1884 Gas 重定价）
2020.12  The Beacon Chain 启动（Phase 0，PoS 共识链上线）
2021.4   Berlin（EIP-2929 访问列表）
2021.8   London ── EIP-1559 费用改革
2022.9   The Merge ── PoW → PoS，执行层与共识层合并
2023.4   Shanghai/Capella ── EIP-4895 质押提款解锁
2024.3   Dencun ── EIP-4844 Proto-Danksharding（Blob 交易）
2025.5   Pectra ── EIP-7702 账户抽象、EIP-7251 验证者合并、EIP-7691 Blob 扩容
2025.12  Fusaka ── EIP-7594 PeerDAS、EIP-7781 出块时间优化（注：EIP-7781 后移）
```

### Pectra（Prague-Electra）完整 EIP 列表

| EIP | 类型 | 主题 |
|-----|------|------|
| EIP-7702 | 执行层 | EOA 账户代码 |
| EIP-7691 | 执行层 | Blob 吞吐量提升 |
| EIP-7623 | 执行层 | Calldata 成本地板价 |
| EIP-2537 | 执行层 | BLS12-381 预编译 |
| EIP-2935 | 执行层 | 历史区块哈希存入状态 |
| EIP-7685 | 执行层 | 通用执行层请求框架 |
| EIP-7251 | 共识层 | 最大有效余额 2048 ETH |
| EIP-7002 | 共识层 | 执行层触发退出 |
| EIP-6110 | 共识层 | 链上提供验证者存款 |
| EIP-7549 | 共识层 | Attestation 委员会索引外移 |
| EIP-7840 | 信息类 | Blob 调度标准化 |

### Fusaka（Fulu-Osaka）完整 EIP 列表

| EIP | 类型 | 主题 |
|-----|------|------|
| EIP-7594 | 共识层/网络 | PeerDAS 点对点数据可用性采样 |
| EIP-7951 | 执行层 | secp256r1 曲线预编译（P-256） |
| EIP-7939 | 执行层 | CLZ（前导零计数）操作码 |
| EIP-7934 | 执行层 | RLP 执行区块大小限制（10 MiB） |
| EIP-7918 | 执行层 | Blob base fee 与执行 gas 耦合地板价 |
| EIP-7917 | 共识层 | 确定性 Proposer 前瞻 |
| EIP-7883 | 执行层 | ModExp Gas 成本增加 |
| EIP-7825 | 执行层 | 单交易 Gas 上限（2²⁴ ≈ 16.78M） |
| EIP-7823 | 执行层 | MODEXP 输入位数限制（8192 bit） |
| EIP-7892 | 信息类 | BPO（仅 Blob 参数）分叉机制 |
| EIP-7642 | 网络层 | eth/69：丢弃 pre-Merge 字段 |
| EIP-7910 | 接口 | eth_config JSON-RPC 方法 |

### Fusaka 之后：BPO 分叉路线

Fusaka 引入了 **BPO（Blob Parameter Only）分叉** 机制，允许在无需完整协议升级的情况下调整 blob 参数：

```
Fusaka 激活（2025.12）：blob target 6 → 10，max 9 → 15
BPO-2（计划中）       ：blob target 10 → 14，max 15 → 21
```

---

## 未来路线图

### Vitalik 路线图六大支柱（截至 2025）

| 支柱 | 关键技术 | 状态 |
|------|---------|------|
| **The Merge** | PoS 共识 | 已完成 |
| **The Surge** | Full Danksharding、PeerDAS、Rollup 扩容 | 进行中（PeerDAS 已上线） |
| **The Scourge** | PBS、MEV 治理、去中心化质押 | 研究中 |
| **The Verge** | Verkle Trees、无状态客户端 | 开发中 |
| **The Purge** | 历史数据清理、协议简化 | 部分实施（EIP-4444） |
| **The Splurge** | EVM 改进（EOF）、账户抽象完善 | 长期演进 |

### 近期重要技术节点

- **Verkle Trees**：替换 MPT，witness 从 ~MB 级降至 ~KB 级，实现无状态客户端
- **Full Danksharding**：64+ blobs/block，结合 PeerDAS 实现 100x+ 扩容
- **Single Slot Finality（SSF）**：将最终确认从 ~12-15 分钟缩短至单个 slot（12 秒）
- **PBS 正式化**：Proposer-Builder Separation 内嵌协议，减少 MEV 中心化风险
- **EOF（EVM Object Format）**：EVM 字节码格式化，提升安全性与可升级性

---

## 上级目录

- [chain-dev/layer1/eip-and-upgrades.md](../../eip-and-upgrades.md) — EIP 速览索引
- [chain-dev/layer1/README.md](../../README.md) — Layer 1 总览

## 参考资源

- [以太坊官方路线图](https://ethereum.org/en/roadmap/)
- [EIPs 官方仓库](https://eips.ethereum.org/)
- [EIP-7607: Hardfork Meta - Fusaka](https://eips.ethereum.org/EIPS/eip-7607)
- [EIP-7600: Hardfork Meta - Pectra](https://eips.ethereum.org/EIPS/eip-7600)
- [AllCoreDevs 会议记录](https://github.com/ethereum/pm)
- [Ethereum Magicians 论坛](https://ethereum-magicians.org/)
