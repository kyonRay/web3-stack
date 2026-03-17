# 深度安全与审计

> **标签：** 安全、重入、flash-loan、mev、审计、slither

本文深入 Solidity 合约安全，超越基础 CEI 模式，覆盖重入变种、Flash Loan 攻击、MEV、价格操纵、以及专业审计流程。

---

## 1. 重入攻击变种

### 1.1 经典重入

```solidity
// 攻击者合约
contract Attacker {
    IVault vault;
    uint256 attackCount;

    receive() external payable {
        if (attackCount < 10 && address(vault).balance >= 1 ether) {
            attackCount++;
            vault.withdraw(1 ether); // 递归调用
        }
    }

    function attack() external {
        vault.deposit{value: 1 ether}();
        vault.withdraw(1 ether); // 触发 receive → 再次 withdraw
    }
}
```

### 1.2 跨函数重入

```solidity
// 合约有两个函数共享状态
mapping(address => uint256) balances;
bool locked;

function withdraw() external {
    require(!locked);
    locked = true;
    (bool s,) = msg.sender.call{value: balances[msg.sender]}("");
    // 此处被重入进入 transfer()，balances 尚未清零
    balances[msg.sender] = 0;
    locked = false;
}

function transfer(address to, uint256 amount) external {
    require(balances[msg.sender] >= amount);
    balances[to] += amount;
    balances[msg.sender] -= amount;
}
```

### 1.3 只读重入（Read-Only Reentrancy）

```solidity
// 某协议在 ETH 发送时更新状态，但 view 函数在发送期间被调用
// 导致其他协议读到不一致状态

// 示例：Curve LP 价格预言机漏洞（2023 年多个协议受影响）
// get_virtual_price() 在 remove_liquidity() 执行中被外部协议调用
// 读到的 virtual_price 是临时高值，造成误判
```

**防御**：对外部 view 调用也要警惕重入；使用 `nonReentrant` 修饰符时考虑是否覆盖所有路径。

---

## 2. Oracle 操纵与价格攻击

### 2.1 现货价格操纵

```solidity
// 危险：直接使用 Uniswap 现货储量计算价格
(uint112 r0, uint112 r1,) = pair.getReserves();
uint256 price = r1 / r0; // 可被 Flash Loan 操纵！

// 安全：使用 TWAP（时间加权平均价格）
uint32 twapInterval = 1800; // 30 分钟
(int24 tick,) = OracleLibrary.consult(pool, twapInterval);
uint256 price = OracleLibrary.getQuoteAtTick(tick, 1e18, tokenIn, tokenOut);
```

### 2.2 Flash Loan 攻击流程

```
1. 攻击者借出大量 TokenA（Flash Loan，无抵押）
2. 在 DEX 大量卖出 TokenA，拉低现货价格
3. 触发目标协议清算（因现货价格显示抵押不足）
4. 获利后归还 Flash Loan
全程在单笔交易内完成
```

**防御措施**：
- 使用 Chainlink 等外部预言机（带延迟，不可单块操纵）
- 使用 TWAP（时间窗口足够长）
- 结合多源价格

---

## 3. MEV（最大可提取价值）

MEV 是验证者/搜索者通过控制交易排序获取的额外收益：

| MEV 类型 | 说明 |
|---------|------|
| **三明治攻击** | 在用户交易前后各插一笔，拉高再拉低价格 |
| **套利** | 利用 DEX 间价差，无害 |
| **清算** | 抢先触发健康因子不足的清算，获取奖励 |
| **抢跑（Front-run）** | 复制高利润的待处理交易并设更高 Gas |

**合约层面防御**：

```solidity
// 1. 滑点保护（DEX 交互必须有）
function swap(uint256 amountIn, uint256 minAmountOut) external {
    uint256 out = _swap(amountIn);
    require(out >= minAmountOut, "Slippage too high");
}

// 2. Commit-Reveal（防止抢跑知道提交内容）
// Phase 1: 提交哈希
bytes32 commitment = keccak256(abi.encodePacked(choice, secret));
commitments[msg.sender] = commitment;

// Phase 2: 揭示（在截止后）
function reveal(uint8 choice, bytes32 secret) external {
    require(keccak256(abi.encodePacked(choice, secret)) == commitments[msg.sender]);
    // 处理
}

// 3. 使用 Flashbots Protect RPC 发送私有交易（DApp 层）
```

---

## 4. 整数与精度问题

```solidity
// 除法截断导致精度损失
uint256 reward = (userBalance * rewardRate) / PRECISION;
// 如果 userBalance * rewardRate 太小，结果为 0！

// 先乘后除（保持精度）
uint256 reward = (userBalance * rewardRate * 1e18) / (PRECISION * 1e18);

// 定点数精度（推荐使用 FixedPoint 库或 PRBMath）
import "prb-math/PRBMathUD60x18.sol";

// 极端情况：除数为零
require(totalSupply > 0, "No supply");
uint256 share = amount * 1e18 / totalSupply;
```

---

## 5. 签名验证安全

```solidity
// 危险：未检查签名重放
function execute(bytes32 hash, bytes memory sig) external {
    address signer = ECDSA.recover(hash, sig);
    require(signer == owner);
    // ...
}

// 安全：防重放（包含 nonce + chainId）
import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";

bytes32 constant TYPEHASH = keccak256("Execute(address target,uint256 value,uint256 nonce,uint256 deadline)");

function execute(
    address target, uint256 value,
    uint256 deadline, bytes memory sig
) external {
    require(block.timestamp <= deadline, "Expired");
    bytes32 structHash = keccak256(abi.encode(TYPEHASH, target, value, nonces[owner]++, deadline));
    bytes32 hash = _hashTypedDataV4(structHash);
    address signer = ECDSA.recover(hash, sig);
    require(signer == owner, "Invalid sig");
}
```

---

## 6. 外部调用安全

```solidity
// 低级调用返回值必须检查
(bool success, bytes memory data) = target.call{value: amount}(calldata);
require(success, "Call failed");

// SafeERC20：处理不规范 ERC-20（不返回 bool 的 token）
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
IERC20(token).safeTransfer(recipient, amount);   // 比 transfer 更安全
IERC20(token).safeTransferFrom(from, to, amount);

// approve 竞态：先设为 0 再设新值（ERC-20 标准建议）
token.safeApprove(spender, 0);
token.safeApprove(spender, newAmount);
// 或改用 permit（ERC-2612）
```

---

## 7. 自动化审计工具

| 工具 | 类型 | 说明 |
|------|------|------|
| **Slither** | 静态分析 | Trail of Bits 开发，检测 80+ 类漏洞 |
| **Aderyn** | 静态分析 | Cyfrin 开发，Rust 实现，速度快 |
| **Mythril** | 符号执行 | ConsenSys 开发，深度分析 |
| **Echidna** | 模糊测试 | Trail of Bits，基于属性的 fuzzing |
| **Medusa** | 模糊测试 | Echidna 继任者，更快 |
| **Manticore** | 符号执行 | Trail of Bits，支持多合约分析 |

```bash
# Slither 快速扫描
slither . --config-file slither.config.json

# Foundry 内置模糊测试
forge test --fuzz-runs 10000
```

---

## 8. 审计流程建议

1. **自查**：跑 Slither/Aderyn，修复低悬果实漏洞
2. **内部审查**：团队代码审查，聚焦业务逻辑
3. **第三方审计**：选有 Web3 专业背景的机构（Certik、Trail of Bits、Spearbit 等）
4. **Bug Bounty**：上线后在 Immunefi 设悬赏
5. **持续监控**：Tenderly Alert、Chainalysis 等监控链上异常

---

## 参考资源

- [Slither](https://github.com/crytic/slither)
- [Aderyn](https://github.com/Cyfrin/aderyn)
- [Immunefi Bug Bounty](https://immunefi.com/)
- [Solodit 审计报告数据库](https://solodit.xyz/)
- [DeFi Hack Labs（历史事件）](https://github.com/SunWeb3Sec/DeFiHackLabs)
