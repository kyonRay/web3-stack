# 以太坊 EIP 与升级路线

> **标签：** eip、eip-1559、eip-4844、eip-7702、账户抽象、以太坊路线图

以太坊通过 **EIP（Ethereum Improvement Proposal）** 持续演进，涵盖协议规则、接口标准与信息类改进。本文聚焦影响链开发的重要 EIP 与主网升级路线。

---

## 1. 关键 EIP 速览

| EIP | 名称 | 状态 | 影响 |
|-----|------|------|------|
| EIP-1559 | 费用市场改革 | 已上线（London） | base fee 自动调整 + 销毁 |
| EIP-4844 | Proto-Danksharding | 已上线（Dencun） | blob 数据类型，大幅降低 L2 DA 成本 |
| EIP-4337 | 账户抽象（无协议修改） | 已上线（合约级） | 智能合约账户、Bundler |
| EIP-7702 | 账户抽象（协议级） | 已上线（Pectra） | EOA 临时委托代码执行 |
| EIP-3074 | AUTH/AUTHCALL | 被 EIP-7702 替代 | — |
| EIP-2537 | BLS12-381 预编译 | 计划中 | ZK 友好签名聚合 |
| EIP-4895 | 信标链提款 | 已上线（Shanghai） | 质押 ETH 可提款 |

---

## 2. EIP-1559（London, 2021）

EIP-1559 重构了以太坊的费用机制：

```
交易费 = (base fee + priority fee) × gas used
base fee  → 每块自动调整（拥堵时升高，空闲时降低），全额销毁
priority fee（小费）→ 给验证者
```

**核心影响：**
- ETH 在高拥堵期净销毁（通缩）
- 费用可预测性提升，减少竞价失败
- 对合约开发：`tx.gasprice` 仍可用，但语义变为 `base fee + priority fee`

---

## 3. EIP-4844（Dencun, 2024）

Proto-Danksharding 引入 **Blob 数据**（Binary Large Object）：

```
普通交易：calldata 永久存储在链上 → 贵
Blob 交易：blob 数据只保留约 18 天 → 便宜（L2 发布 DA 用）
```

| 参数 | 值 |
|------|-----|
| 每块目标 blob 数 | 3 |
| 每块最大 blob 数 | 6 |
| 每 blob 大小 | 128 KB |
| blob 保留时长 | ~18 天 |

**对 L2 开发的影响：**
- OP Stack/Arbitrum/zkSync 等均已切换 blob 模式提交批次
- L2 DA 成本降低 10-100 倍
- 合约无法直接读取 blob 内容（只能验证 blob 哈希）

```solidity
// blob 哈希可在合约内读取
bytes32 blobHash = blobhash(0); // 第 0 个 blob 的 KZG 承诺哈希
```

---

## 4. EIP-4337（账户抽象）

无需修改 L1 协议，通过 `EntryPoint` 合约实现账户抽象：

```
用户 → UserOperation → Bundler → EntryPoint → 智能账户合约
                                      └── Paymaster（可选：代付 gas）
```

**核心能力：**
- 自定义签名验证（多签、WebAuthn、社交恢复）
- Gas 代付（ERC-20 支付 gas）
- 批量操作（一次 UserOp 多步）
- 会话密钥

> 详细实现见 [smart-contract-dev/evm/solidity/advanced-topics.md](../../smart-contract-dev/evm/solidity/advanced-topics.md)

---

## 5. EIP-7702（Pectra, 2025）

允许 **EOA 临时委托给合约代码**执行，无需永久转换为智能账户：

```
EOA 在交易中附带 authorization list：
  [{ chain_id, address(代码合约), nonce, signature }]

该交易期间，EOA 的代码区域临时指向指定合约
```

**与 EIP-4337 对比：**
| 维度 | EIP-4337 | EIP-7702 |
|------|---------|---------|
| 账户类型 | 新建智能合约账户 | 现有 EOA 临时升级 |
| 改动范围 | 无协议修改（合约级） | 协议级（Pectra 硬分叉） |
| 兼容性 | 需迁移 | 现有 EOA 直接可用 |
| Paymaster | 支持 | 支持 |

---

## 6. 以太坊升级路线图

```
已完成
├── The Merge（2022）— PoW → PoS
├── Shanghai/Capella（2023）— 质押提款
├── Dencun（2024）— EIP-4844 Blob
└── Pectra（2025）— EIP-7702、EIP-7251（合并验证者）

进行中 / 计划中
├── Verkle Trees — 替换 MPT，支持无状态客户端
├── Full Danksharding — 扩展 blob 容量至 64+
├── PBS（Proposer-Builder Separation）正式化
└── Single Slot Finality（SSF）— 提速终局性
```

---

## 7. 以太坊升级跟踪资源

- [Ethereum Roadmap（ethereum.org）](https://ethereum.org/en/roadmap/)
- [EIPs 官方仓库](https://eips.ethereum.org/)
- [AllCoreDevs 会议记录](https://github.com/ethereum/pm)
- [Vitalik 博客](https://vitalik.eth.limo/)
- [Ethereum Magicians（讨论论坛）](https://ethereum-magicians.org/)
