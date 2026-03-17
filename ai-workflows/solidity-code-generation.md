# Solidity 代码生成提示词

> **标签：** ai、提示词、solidity、代码生成、llm

本文提供用于生成、重构 Solidity 代码的提示词，使 AI 输出更贴近生产标准。

> 生成的代码务必在本地编译、运行测试，并做人工安全检查。

---

## 1. ERC-20 代币生成

```
使用 Solidity 0.8.x 和 OpenZeppelin Contracts v5 编写一个可用于生产的 ERC-20 代币。

要求：
- 名称：[NAME]，符号：[SYMBOL]，精度：18
- 初始供应量 [AMOUNT] 铸造给部署者
- 具有 MINTER_ROLE 的地址可铸造
- 持币者可销毁
- 具有 PAUSER_ROLE 的地址可暂停
- 通过 UUPS 代理模式可升级
- 所有对外函数有 NatSpec 注释
- 提供 Foundry 测试，覆盖：铸造、销毁、暂停、角色管理、升级

使用 OpenZeppelin 可升级库，遵循 CEI，使用 custom errors。
```

---

## 2. ERC-721 NFT 生成

```
使用 Solidity 0.8.x 和 OpenZeppelin v5 编写一个可用于生产的 ERC-721 NFT 合约。

要求：
- 名称：[NAME]，符号：[SYMBOL]
- 最大供应量：[MAX_SUPPLY]
- 公开铸造价格：[PRICE] ETH，每钱包最多 [N] 个
- 揭示机制：baseURI 由 owner 可改，默认指向未揭示图
- 揭示前在链上提交 provenance hash
- EIP-2981 版税 [X]% 给部署者
- owner 可提取 ETH
- 使用 ERC721A（批量 mint 优化）
- 含 NatSpec 与 Foundry 测试：mint、批量 mint、钱包限制、揭示、版税、提款

使用 custom errors，所有状态变更发出事件。
```

---

## 3. Vault 质押合约

```
使用 Solidity 0.8.x 编写一个 ERC-20 代币质押 Vault 合约（类似 ERC-4626 简化版）。

要求：
- 用户可以存入 [UNDERLYING_TOKEN]，获得 share 代币
- 支持提款（burn share → 取回资产 + 收益）
- 收益来源：owner 定期向 Vault 注入奖励代币
- share 价格根据 totalAssets / totalShares 计算
- 防重入保护
- 事件：Deposit、Withdraw、RewardAdded
- 提供 Foundry 测试和模糊测试

遵循 CEI，使用 custom errors 和 OpenZeppelin。
```

---

## 4. 多签钱包生成

```
用 Solidity 0.8.x 编写一个最小可用的多签钱包合约。

要求：
- 部署时设定 owners 数组（不可变）
- 可配置确认阈值（如 2-of-3）
- 提交交易：任一 owner 可提议（to, value, calldata）
- 确认：任一 owner 可确认待执行提议
- 执行：达到阈值后任一 owner 可执行
- 撤销：执行前 owner 可撤销自己的确认
- 事件：Submission、Confirmation、Revocation、Execution

安全：防止重复 owner、非 owner 访问、同一人重复确认；执行遵循 CEI。
含 NatSpec 与 Foundry 测试。
```

---

## 5. 代码审查与重构

```
你是一名 Solidity 与 EVM 专家。请对以下代码进行全面审查：

1. 安全：重入、访问控制、整数与精度、签名重放、可升级存储布局
2. Gas：存储打包、SLOAD 缓存、calldata、custom errors、unchecked
3. 代码质量：命名、注释、事件、错误提示的清晰度
4. ERC 规范符合度（如涉及代币标准）

对每个问题：标明严重程度（Critical/High/Medium/Low），给出修改建议或补丁。

代码：
```solidity
[在此粘贴代码]
```
```

---

## 6. 测试生成

```
为以下 Solidity 合约生成完整的 Foundry 测试套件。

测试要求：
1. 为每个公开/外部函数编写正常路径测试
2. 为每个 revert 场景编写负面测试（vm.expectRevert）
3. 为数值型函数添加模糊测试（testFuzz_xxx）
4. 用不变式测试（invariant_xxx）验证核心状态不变式
5. 使用 vm.prank 模拟不同调用者
6. 用 vm.warp/vm.roll 测试时间依赖逻辑

合约代码：
```solidity
[在此粘贴合约]
```
```

---

## 参考资源

- [Solidity 语言深入](../smart-contract-dev/evm/solidity/language-deep-dive.md)
- [安全模式](../smart-contract-dev/evm/solidity/best-practices.md)
- [Gas 优化](../smart-contract-dev/evm/solidity/gas-optimization.md)
- [Foundry Book](https://book.getfoundry.sh/)
