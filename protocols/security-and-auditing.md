# 合约安全与审计

> **标签：** 安全、审计、漏洞、重入、slither、immunefi

智能合约安全是 Web3 开发的核心能力，历史上数十亿美元因漏洞损失。

---

## 1. 常见漏洞类型

| 漏洞 | 严重程度 | 说明 |
|------|---------|------|
| **重入（Reentrancy）** | 严重 | 外部调用前状态未更新，攻击者可递归调用 |
| **访问控制缺失** | 严重 | 特权函数无权限检查 |
| **整数溢出/下溢** | 高 | Solidity 0.8+ 已默认检查，老合约风险 |
| **Oracle 操纵** | 高 | 使用现货价格，可被闪电贷操纵 |
| **Flash Loan 攻击** | 高 | 借大量资金操纵协议状态 |
| **Front-Running（抢跑）** | 中 | 观察 mempool 提前交易 |
| **签名重放** | 高 | 相同签名在不同上下文重用 |
| **存储布局错误** | 严重 | 可升级合约升级后变量错位 |
| **精度损失** | 中 | 除法截断导致计算错误 |

---

## 2. 历史重大事件

| 事件 | 年份 | 损失 | 原因 |
|------|------|------|------|
| The DAO Hack | 2016 | $60M | 重入攻击 |
| Poly Network | 2021 | $611M | 跨链权限绕过 |
| Wormhole | 2022 | $320M | Solana 签名验证漏洞 |
| Ronin Bridge | 2022 | $624M | 多签私钥泄露（社会工程） |
| Euler Finance | 2023 | $200M | 闪电贷 + 清算逻辑漏洞 |
| Curve（只读重入） | 2023 | $73M | 只读重入攻击 |

---

## 3. 安全工具

| 工具 | 类型 |
|------|------|
| **Slither** | 静态分析（Trail of Bits） |
| **Aderyn** | 静态分析（Cyfrin，Rust） |
| **Echidna/Medusa** | 模糊测试（Trail of Bits） |
| **Manticore** | 符号执行（Trail of Bits） |
| **Mythril** | 符号执行（ConsenSys） |
| **Certora Prover** | 形式化验证 |
| **Foundry invariant** | 不变式测试 |

---

## 4. 审计流程

1. **范围界定**：明确合约范围、信任边界、链上 + 链下依赖
2. **文档阅读**：协议设计文档、接口说明、已知风险
3. **自动化扫描**：Slither、Aderyn 等工具
4. **人工审查**：逐函数审查业务逻辑、安全模式
5. **编写报告**：按严重程度分类（Critical/High/Medium/Low/Info）
6. **修复 + 复验**：开发方修复，审计方验证修复有效性
7. **Bug Bounty**：上线后在 Immunefi 设悬赏

---

## 5. 安全开发规范

- 遵循 [smart-contract-dev/evm/solidity/best-practices.md](../smart-contract-dev/evm/solidity/best-practices.md)
- 所有合约通过 Slither 扫描，无 high/medium 未处理项
- 使用 Foundry invariant 测试核心不变式
- 主网大额资金合约：专业审计 + Bug Bounty

---

## 参考资源

- [Slither](https://github.com/crytic/slither)
- [Immunefi Bug Bounty](https://immunefi.com/)
- [Solodit 审计报告数据库](https://solodit.xyz/)
- [DeFi Hack Labs](https://github.com/SunWeb3Sec/DeFiHackLabs)
- [SWC Registry](https://swcregistry.io/)
