# Solidity 精通路径

Solidity 是 EVM 生态最主流的合约语言，精通 Solidity 是成为合约开发专家的核心要求。

## 文档列表

| 文档 | 难度 | 说明 |
|------|------|------|
| [language-deep-dive.md](./language-deep-dive.md) | ⭐⭐ | 语言核心：类型、存储、ABI、继承 |
| [best-practices.md](./best-practices.md) | ⭐⭐⭐ | 安全模式：CEI、访问控制、常见漏洞防御 |
| [gas-optimization.md](./gas-optimization.md) | ⭐⭐⭐ | Gas 优化：存储打包、SLOAD 缓存、unchecked |
| [security-patterns.md](./security-patterns.md) | ⭐⭐⭐⭐ | 深度安全：重入变种、Flash Loan、MEV、审计 |
| [advanced-topics.md](./advanced-topics.md) | ⭐⭐⭐⭐ | 进阶：Create2、Delegatecall、代理、账户抽象 |

## 学习路径

```
语言基础 → 安全模式 → Gas 优化
    ↓
  深度安全 → 进阶主题
    ↓
  项目实战（DeFi / NFT / DAO 合约）
```

## 必须掌握的核心主题

- [ ] Solidity 类型系统（value types、reference types、函数类型）
- [ ] 存储布局（storage、memory、calldata、stack 的区别）
- [ ] ABI 编码（abi.encode、abi.encodePacked、selector）
- [ ] Checks-Effects-Interactions 模式
- [ ] OpenZeppelin 标准库（Ownable、AccessControl、ReentrancyGuard）
- [ ] ERC-20/721/1155 实现细节
- [ ] Gas 优化技巧（SSTORE 打包、缓存、unchecked）
- [ ] 代理模式（Transparent、UUPS）
- [ ] Create2 与 Delegatecall

## 参考资源

- [Solidity 官方文档](https://docs.soliditylang.org/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Foundry Book](https://book.getfoundry.sh/)
- [Solidity By Example](https://solidity-by-example.org/)
- [Cyfrin Updraft（免费课程）](https://updraft.cyfrin.io/)
