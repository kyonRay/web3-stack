# 链上治理

> **标签：** 治理、governor、投票、提案、timelock

链上治理允许协议参数、升级决策通过透明的投票流程执行，而非中心化控制。

---

## 1. Governor 合约完整流程

```
1. 提案（Propose）
   持有足够治理代币（≥提案阈值）的用户创建提案
   提案包含：描述 + 执行的合约调用（to/value/calldata）

2. 投票延迟（Voting Delay）
   等待 N 个区块后投票开始（让 holder 有时间准备）

3. 投票期（Voting Period）
   holder 通过 castVote() 投票（赞成/反对/弃权）

4. 排队（Queue）
   达到法定人数且赞成票多时，提案进入 Timelock 队列

5. 执行（Execute）
   Timelock 延迟到期后，任何人可调用 execute()
```

---

## 2. 委托（Delegation）

```solidity
// ERC20Votes 代币委托
// 自委托（让自己的代币有投票权）
token.delegate(msg.sender);

// 委托给他人
token.delegate(delegateeAddress);

// 使用 delegateBySig 无 gas 委托（EIP-712 签名）
token.delegateBySig(delegatee, nonce, expiry, v, r, s);
```

**重要**：ERC20Votes 代币只有在**委托后**才有投票权，未委托的代币无投票权。

---

## 3. 治理攻击防护

| 攻击 | 说明 | 防护 |
|------|------|------|
| **Flash Loan 治理** | 借入大量代币提案或投票 | 使用过去区块的快照投票（ERC20Votes）|
| **投票操纵** | 大户否决社区意见 | 法定人数 + 时间锁 |
| **提案攻击** | 恶意提案升级合约 | Timelock 给社区时间发现并应对 |

---

## 参考资源

- [OpenZeppelin Governor](https://docs.openzeppelin.com/contracts/governance)
- [Compound Governor Bravo](https://compound.finance/governance)
- [Uniswap Governance](https://gov.uniswap.org/)
