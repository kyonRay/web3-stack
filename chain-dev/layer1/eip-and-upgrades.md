# 以太坊 EIP 与升级路线

> **标签：** eip、eip-1559、eip-4844、eip-7702、账户抽象、以太坊路线图

以太坊通过 **EIP（Ethereum Improvement Proposal）** 持续演进，涵盖协议规则、接口标准与信息类改进。本文为关键 EIP 速览索引，详细分析请查阅 [eth/eip/](./eth/eip/) 目录下的专项文档。

---

## 关键 EIP 速览

| EIP | 名称 | 升级 | 状态 | 详细文档 |
|-----|------|------|------|---------|
| EIP-1559 | 费用市场改革（base fee + 燃烧） | London（2021.8） | 已上线 | [eip-1559-fee-market.md](./eth/eip/eip-1559-fee-market.md) |
| EIP-4895 | 信标链质押提款 | Shanghai（2023.4） | 已上线 | [eip-4895-beacon-withdrawals.md](./eth/eip/eip-4895-beacon-withdrawals.md) |
| EIP-4844 | Proto-Danksharding（Blob 交易） | Dencun（2024.3） | 已上线 | [eip-4844-proto-danksharding.md](./eth/eip/eip-4844-proto-danksharding.md) |
| EIP-4337 | 账户抽象（合约级，无协议修改） | 独立部署 | 已上线 | [eip-4337-account-abstraction.md](./eth/eip/eip-4337-account-abstraction.md) |
| EIP-7702 | EOA 账户代码委托 | Pectra（2025.5） | 已上线 | [eip-7702-eoa-code.md](./eth/eip/eip-7702-eoa-code.md) |
| EIP-7251 | 验证者最大有效余额 32→2048 ETH | Pectra（2025.5） | 已上线 | [eip-7251-max-effective-balance.md](./eth/eip/eip-7251-max-effective-balance.md) |
| EIP-7691 | Blob 吞吐量翻倍（target 3→6） | Pectra（2025.5） | 已上线 | [eip-7691-blob-throughput.md](./eth/eip/eip-7691-blob-throughput.md) |
| EIP-7623 | Calldata 地板价提升 | Pectra（2025.5） | 已上线 | [eip-7623-calldata-cost.md](./eth/eip/eip-7623-calldata-cost.md) |
| EIP-2537 | BLS12-381 曲线预编译 | Pectra（2025.5） | 已上线 | [eip-2537-bls-precompile.md](./eth/eip/eip-2537-bls-precompile.md) |
| EIP-7594 | PeerDAS 数据可用性采样 | Fusaka（2025.12） | 已上线 | [eip-7594-peerdas.md](./eth/eip/eip-7594-peerdas.md) |
| EIP-3074 | AUTH/AUTHCALL（已废弃） | — | 被 EIP-7702 替代 | — |

---

## 以太坊升级时间线

```
2022.9   The Merge ── PoW → PoS，执行层与共识层合并
2023.4   Shanghai/Capella ── EIP-4895 质押提款解锁
2024.3   Dencun ── EIP-4844 Proto-Danksharding（Blob 交易）
2025.5   Pectra ── EIP-7702/7251/7691/7623/2537 等
2025.12  Fusaka ── EIP-7594 PeerDAS、secp256r1 预编译等
```

---

## EIP 详细文档目录

完整的 EIP 深度解析、前后对比与影响分析请查阅：

**[eth/eip/README.md](./eth/eip/README.md)** — EIP 详解目录（含升级路线图至 Fusaka、未来路线）

---

## 以太坊未来路线简述

```
进行中 / 计划中
├── Fusaka 后 BPO 分叉 — 无需硬分叉调整 blob 参数（target 10→14）
├── Verkle Trees — 替换 MPT，支持无状态客户端
├── Full Danksharding — 扩展 blob 容量至 64+，结合 PeerDAS
├── PBS（Proposer-Builder Separation）正式化
└── Single Slot Finality（SSF）— 终局性从 ~15 分钟缩短至 12 秒
```

---

## 参考资源

- [以太坊官方路线图](https://ethereum.org/en/roadmap/)
- [EIPs 官方仓库](https://eips.ethereum.org/)
- [AllCoreDevs 会议记录](https://github.com/ethereum/pm)
- [Vitalik 博客](https://vitalik.eth.limo/)
- [Ethereum Magicians 论坛](https://ethereum-magicians.org/)
