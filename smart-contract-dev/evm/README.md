# EVM 合约开发（重点精通）

EVM（Ethereum Virtual Machine）生态是目前智能合约最主流的平台，覆盖以太坊主网与众多 L2（Arbitrum、Optimism、Base、zkSync、Polygon 等）及其他 EVM 兼容链。

## 文档列表

### Solidity（核心精通路径）

| 文档 | 说明 |
|------|------|
| [solidity/README.md](./solidity/README.md) | Solidity 学习路径总览 |
| [solidity/language-deep-dive.md](./solidity/language-deep-dive.md) | 语言深入：类型系统、存储模型、ABI 编码 |
| [solidity/best-practices.md](./solidity/best-practices.md) | 安全模式：CEI、访问控制、常见漏洞 |
| [solidity/gas-optimization.md](./solidity/gas-optimization.md) | Gas 优化：存储打包、SLOAD 缓存、位图等 |
| [solidity/security-patterns.md](./solidity/security-patterns.md) | 深度安全：重入变种、flash loan 防护、MEV |
| [solidity/advanced-topics.md](./solidity/advanced-topics.md) | 进阶：Create2、Delegatecall、代理模式、账户抽象 |

### EVM 生态（语言/框架/标准）

| 文档 | 说明 |
|------|------|
| [vyper-and-others.md](./vyper-and-others.md) | Vyper、Yul、Fe、Cairo 等 EVM 语言 |
| [frameworks.md](./frameworks.md) | Foundry、Hardhat、Remix、OpenZeppelin、安全工具（Slither/Echidna） |
| [token-standards.md](./token-standards.md) | ERC-20/721/1155/2612/4626/165/712/1271 等代币与接口标准 |
| [testing-and-auditing.md](./testing-and-auditing.md) | 测试框架、审计流程、Slither/静态分析 |
| [deployment.md](./deployment.md) | 部署脚本、验证、可升级合约、多链部署 |
| [evm-internals.md](./evm-internals.md) | EVM 内部原理：字节码、操作码、内存模型 |
| [evm-compatible-chains.md](./evm-compatible-chains.md) | EVM 兼容链：BNB Chain、Polygon、Avalanche C-Chain、HyperLiquid |

## 推荐学习顺序

**入门阶段**
1. [solidity/language-deep-dive.md](./solidity/language-deep-dive.md) — 扎实语言基础
2. [frameworks.md](./frameworks.md) — 搭建开发环境（推荐 Foundry）
3. [token-standards.md](./token-standards.md) — 实现 ERC-20/721

**进阶阶段**
4. [solidity/best-practices.md](./solidity/best-practices.md) — 安全模式
5. [solidity/gas-optimization.md](./solidity/gas-optimization.md) — 性能优化
6. [testing-and-auditing.md](./testing-and-auditing.md) — 测试覆盖与审计

**精通阶段**
7. [solidity/security-patterns.md](./solidity/security-patterns.md) — 深度安全
8. [solidity/advanced-topics.md](./solidity/advanced-topics.md) — 代理/AA/Create2
9. [evm-internals.md](./evm-internals.md) — EVM 底层原理

## 参考资源

- [Solidity 官方文档](https://docs.soliditylang.org/)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [Foundry Book](https://book.getfoundry.sh/)
- [SWC Registry](https://swcregistry.io/)
