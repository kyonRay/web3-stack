# Solana 合约开发（SVM 生态）

Solana 采用独特的**账户模型**与并行执行的 **Sealevel VM（SVM）**，合约使用 **Rust** 编写，**Anchor** 框架大幅简化开发体验。

## 文档列表

| 文档 | 说明 |
|------|------|
| [rust-for-solana.md](./rust-for-solana.md) | Solana 合约开发所需的 Rust 核心知识 |
| [anchor-framework.md](./anchor-framework.md) | Anchor 框架：账户约束、IDL、宏系统 |
| [spl-tokens.md](./spl-tokens.md) | SPL Token / Token-2022 标准与扩展 |
| [pda-and-cpi.md](./pda-and-cpi.md) | PDA（程序派生地址）与 CPI（跨程序调用） |
| [testing-and-deployment.md](./testing-and-deployment.md) | 测试（localnet/devnet）与部署流程 |
| [svm-internals.md](./svm-internals.md) | SVM 深入：并行执行、账户锁、计算单元 |

## 推荐学习顺序

1. [rust-for-solana.md](./rust-for-solana.md) — Rust 所有权与生命周期基础
2. [anchor-framework.md](./anchor-framework.md) — Anchor 搭建第一个程序
3. [pda-and-cpi.md](./pda-and-cpi.md) — PDA 是 Solana 核心，必须掌握
4. [spl-tokens.md](./spl-tokens.md) — 代币标准
5. [testing-and-deployment.md](./testing-and-deployment.md) — 测试与上链
6. [svm-internals.md](./svm-internals.md) — 深入理解性能与限制

## 与 EVM 的核心差异

| 维度 | EVM | Solana |
|------|-----|--------|
| 状态存储 | 合约内 storage | 独立 Account（租金模型） |
| 合约/程序 | 有状态 | 无状态（状态在 Account） |
| 并行执行 | 顺序（同 block） | Sealevel 并行（无冲突账户） |
| 代币标准 | ERC-20（各自部署） | SPL（统一 Token Program） |
| 开发语言 | Solidity/Vyper | Rust（Anchor） |

## 参考资源

- [Solana 开发者文档](https://docs.solana.com/)
- [Anchor 官方文档](https://www.anchor-lang.com/)
- [SPL Token 文档](https://spl.solana.com/token)
- [Solana Cookbook](https://solanacookbook.com/)
