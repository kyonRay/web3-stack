# DAO 与链上治理

> **标签：** dao、治理、governor、timelock、多签

DAO（去中心化自治组织）通过链上提案与投票决定协议参数调整、资金使用与升级等重大决策。

---

## 1. 治理架构

```
治理代币持有者
    ↓ 创建/委托投票
Governor 合约（提案 + 投票）
    ↓ 通过后排队
Timelock Controller（延迟执行）
    ↓ 到期执行
目标合约（Treasury/Protocol）
```

### 关键参数

| 参数 | 说明 | 典型值 |
|------|------|--------|
| **提案阈值** | 提案所需最少持币量 | 0.1% - 1% 总量 |
| **投票延迟** | 提案创建到投票开始 | 1-2 天 |
| **投票期** | 投票开放时长 | 3-7 天 |
| **法定人数** | 通过所需最少投票量 | 4% - 10% 总量 |
| **Timelock 延迟** | 通过到执行的等待期 | 2-7 天 |

---

## 2. OpenZeppelin Governor

```solidity
import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorSettings.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorCountingSimple.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract MyGovernor is
    Governor,
    GovernorSettings,
    GovernorCountingSimple,
    GovernorVotes,
    GovernorTimelockControl
{
    constructor(IVotes _token, TimelockController _timelock)
        Governor("MyGovernor")
        GovernorSettings(1 days, 1 weeks, 1000e18) // 延迟/期限/阈值
        GovernorVotes(_token)
        GovernorTimelockControl(_timelock)
    {}

    function quorum(uint256 blockNumber) public pure override returns (uint256) {
        return 100_000e18; // 10万票法定人数
    }
}
```

---

## 3. 多签（Safe/Gnosis Safe）

多签是最常见的"DAO 国库"管理方式：

```
安全团队使用 Safe（Gnosis Safe）：
- 3/5 多签：5 个签名者中 3 个同意才能执行
- 支持任意合约调用（不只是转账）
- Safe{Core} SDK 可编程创建多签交易
```

```typescript
import Safe, { EthersAdapter } from '@safe-global/protocol-kit'

const safeSDK = await Safe.create({ ethAdapter, safeAddress })

// 创建提案交易
const safeTransaction = await safeSDK.createTransaction({
    transactions: [{
        to: protocolAddress,
        value: '0',
        data: encodedUpgradeCall,
    }],
})

// 签名
const signedTx = await safeSDK.signTransaction(safeTransaction)

// 广播（收集到足够签名后）
const result = await safeSDK.executeTransaction(signedTx)
```

---

## 4. 委托投票

大多数 DAO 治理代币支持**委托（Delegation）**：持币者将投票权委托给有能力参与治理的代理人。

```solidity
// ERC20Votes 扩展（OpenZeppelin）
token.delegate(delegatee);  // 将投票权委托给 delegatee
uint256 votes = token.getVotes(address); // 当前投票权
uint256 pastVotes = token.getPastVotes(address, blockNumber); // 历史投票权
```

---

## 5. Snapshot（链下投票）

Snapshot 是最广泛使用的**链下投票**工具，持币快照 + IPFS 存储结果，无需 Gas：

- 投票基于历史区块快照的余额，无需链上 Gas
- 投票结果存储在 IPFS，不可篡改
- 支持 ERC-20、NFT、LP 代币多种投票权计算
- 通过 Snapshot X 可与链上 Timelock 集成（链下投票 + 链上执行）

```typescript
// 读取 Snapshot 提案
const response = await fetch('https://hub.snapshot.org/graphql', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: `{
      proposals(where: { space: "uniswap.eth", state: "active" }) {
        id title body start end state scores
      }
    }`
  })
})
```

---

## 参考资源

- [OpenZeppelin Governor 文档](https://docs.openzeppelin.com/contracts/governance)
- [Safe（多签）文档](https://docs.safe.global/)
- [Tally（DAO 界面）](https://www.tally.xyz/)
- [Snapshot 文档](https://docs.snapshot.org/)
- [Snapshot X（链下投票 + 链上执行）](https://snapshotx.org/)
