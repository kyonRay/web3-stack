# Move 合约开发（Sui / Aptos）

Move 是一种**资源型（Resource-Oriented）**语言，起源于 Facebook Diem 项目，现被 Sui 和 Aptos 采用。其核心创新是将数字资产建模为**不可复制、不可丢弃**的资源，从语言层面防止一类安全问题。

## 文档列表

| 文档 | 说明 |
|------|------|
| [move-language.md](./move-language.md) | Move 语言核心：类型系统、能力（abilities）、资源模型 |
| [sui-development.md](./sui-development.md) | Sui Move：对象模型、动态字段、Kiosk、PTB |
| [aptos-development.md](./aptos-development.md) | Aptos Move：模块系统、资源账户、框架差异 |
| [testing-and-deployment.md](./testing-and-deployment.md) | 测试（Move Test）与链上部署 |

## 推荐学习顺序

1. [move-language.md](./move-language.md) — 理解 Move 的资源模型（与 Solidity 最大差异）
2. 按目标链选择：[sui-development.md](./sui-development.md) 或 [aptos-development.md](./aptos-development.md)
3. [testing-and-deployment.md](./testing-and-deployment.md) — 测试与发布

## Sui vs Aptos 差异

| 维度 | Sui Move | Aptos Move |
|------|----------|-----------|
| 状态模型 | 以对象（Object）为核心，全局存储为对象集合 | 以账户（Account）为核心，资源在账户下 |
| 并行执行 | 对象级并行（无冲突对象可并行） | Block-STM（乐观并发） |
| 特有特性 | PTB（可编程交易块）、Kiosk、动态字段 | 资源账户、功能（functions）框架 |
| 代币标准 | Coin<T>，NFT 可选 Kiosk | Fungible Asset（新标准）、token v2 |

## 核心概念：Move 的四种能力（Abilities）

```
copy    — 值可被复制
drop    — 值可被丢弃（不必显式使用）
store   — 值可存入全局存储
key     — 值可作为全局存储的顶层键（Sui 中：作为 Object）
```

普通代币/NFT 通常只有 `store` 和 `key`，无 `copy` 和 `drop`，从而保证唯一性与不可凭空销毁。

## 参考资源

- [Move Book](https://move-book.com/)
- [Sui 开发者文档](https://docs.sui.io/)
- [Aptos 开发者文档](https://aptos.dev/)
- [Move 语言规范](https://github.com/move-language/move)
