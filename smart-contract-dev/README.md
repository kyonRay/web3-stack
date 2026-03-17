# 合约开发专家

本方向面向希望精通区块链智能合约开发的工程师，覆盖三大主流生态：EVM（以 Solidity 为核心）、Solana（SVM + Rust/Anchor）和 Move（Sui/Aptos）。

## 方向概览

| 子目录 | 生态 | 核心语言/框架 | 说明 |
|--------|------|--------------|------|
| [evm/](./evm/) | EVM 生态 | Solidity / Vyper / Yul | **重点精通方向**，覆盖以太坊主网及所有 EVM 兼容链 |
| [solana/](./solana/) | Solana/SVM | Rust / Anchor | 高性能合约开发，PDA、CPI 等核心机制 |
| [move/](./move/) | Move 生态 | Move | Sui 与 Aptos 的资源型语言与对象模型 |

## 选择方向

```
合约开发
├── EVM（推荐首选）
│   ├── 受众最广，工具链最成熟
│   ├── 以太坊主网 + L2（Arbitrum/Optimism/Base/zkSync 等）
│   └── 语言：Solidity（精通）、Vyper/Yul（了解）
├── Solana
│   ├── 高性能，适合高频 DeFi / NFT
│   ├── 独特账户模型，需要适应
│   └── 语言：Rust（Anchor 框架简化开发）
└── Move
    ├── 资源安全，对象模型防重入
    ├── Sui / Aptos 生态
    └── 语言：Move（Sui Move / Aptos Move 略有差异）
```

## 通用技能

无论选择哪个生态，以下能力都是核心：

- 合约安全意识（重入、访问控制、整数精度等）
- 测试驱动开发（单元测试、模糊测试、形式化验证）
- Gas/费用优化思维
- 审计与代码审查流程

## 延伸阅读

- [protocols/](../protocols/) — DeFi、NFT、DAO 等合约应用场景
- [ai-workflows/](../ai-workflows/) — AI 辅助合约审计与代码生成
