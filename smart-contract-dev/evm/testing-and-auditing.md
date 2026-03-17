# 测试、审计与验证

> **标签：** 测试、foundry、模糊测试、不变式测试、slither、审计

合约不可篡改，漏洞代价极高。测试与审计是降低风险的主要手段，应该贯穿开发全程。

---

## 1. 测试策略分层

```
不变式测试（Invariant Testing）  — 最高层，验证系统属性
模糊测试（Fuzz Testing）         — 自动生成输入，发现边界情况
集成测试（Integration Testing）  — 多合约交互、fork 主网场景
单元测试（Unit Testing）         — 单函数逻辑验证
```

---

## 2. Foundry 测试实践

### 2.1 单元测试

```solidity
// test/Vault.t.sol
pragma solidity ^0.8.20;
import "forge-std/Test.sol";
import "../src/Vault.sol";

contract VaultTest is Test {
    Vault vault;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        vault = new Vault();
        vm.deal(alice, 10 ether);
    }

    function test_Deposit() public {
        vm.prank(alice);
        vault.deposit{value: 1 ether}();
        assertEq(vault.balances(alice), 1 ether);
    }

    function test_Withdraw_RevertIfInsufficient() public {
        vm.prank(alice);
        vm.expectRevert(Vault.InsufficientBalance.selector);
        vault.withdraw(1 ether);
    }

    function test_WithdrawAll() public {
        vm.startPrank(alice);
        vault.deposit{value: 1 ether}();
        uint256 balanceBefore = alice.balance;
        vault.withdraw(1 ether);
        assertEq(alice.balance, balanceBefore + 1 ether);
        vm.stopPrank();
    }
}
```

### 2.2 模糊测试

```solidity
// Foundry 自动生成随机输入（默认 256 次，可用 --fuzz-runs 调整）
function testFuzz_DepositWithdraw(uint128 amount) public {
    vm.assume(amount > 0 && amount <= 100 ether);
    vm.deal(alice, amount);

    vm.prank(alice);
    vault.deposit{value: amount}();
    assertEq(vault.balances(alice), amount);

    vm.prank(alice);
    vault.withdraw(amount);
    assertEq(vault.balances(alice), 0);
}
```

### 2.3 不变式测试（最强大）

```solidity
// test/Vault.invariant.t.sol
contract VaultInvariantTest is Test {
    Vault vault;
    VaultHandler handler;

    function setUp() public {
        vault = new Vault();
        handler = new VaultHandler(vault);
        targetContract(address(handler)); // 让 Foundry 随机调用 handler 的函数
    }

    // 不变式：vault 的 ETH 余额 >= 所有用户存款之和
    function invariant_SolvencyCheck() public {
        assertGe(address(vault).balance, handler.getTotalDeposits());
    }

    // 不变式：totalSupply 不会凭空增加
    function invariant_TotalSupply() public {
        assertEq(vault.totalShares(), handler.sumShares());
    }
}

// Handler 封装合法操作
contract VaultHandler is Test {
    Vault vault;
    uint256 public totalDeposits;
    address[] actors;

    constructor(Vault _vault) {
        vault = _vault;
        for (uint i = 0; i < 5; i++) {
            actors.push(makeAddr(string(abi.encodePacked("actor", i))));
        }
    }

    function deposit(uint256 actorSeed, uint256 amount) external {
        amount = bound(amount, 0.001 ether, 10 ether);
        address actor = actors[actorSeed % actors.length];
        vm.deal(actor, amount);
        vm.prank(actor);
        vault.deposit{value: amount}();
        totalDeposits += amount;
    }
}
```

### 2.4 常用 Cheatcodes

```solidity
vm.prank(address)          // 下一次调用的 msg.sender
vm.startPrank / stopPrank  // 区间内的 msg.sender
vm.deal(addr, amount)      // 设置 ETH 余额
vm.warp(timestamp)         // 设置 block.timestamp
vm.roll(blockNumber)       // 设置 block.number
vm.expectRevert(selector)  // 期望下一次调用 revert
vm.expectEmit(...)         // 期望下一次调用发出特定事件
vm.mockCall(addr, data, returnData) // mock 合约调用
vm.createSelectFork("mainnet")      // fork 主网
```

---

## 3. 静态分析工具

### 3.1 Slither

```bash
# 安装
pip install slither-analyzer

# 基础扫描
slither .

# 针对特定检测器
slither . --detect reentrancy-eth,unprotected-upgrade

# 生成报告
slither . --json slither-report.json
```

常见检测项：

| 检测器 | 说明 |
|--------|------|
| `reentrancy-eth` | ETH 重入 |
| `reentrancy-no-eth` | 非 ETH 重入（状态不一致） |
| `unprotected-upgrade` | 无权限检查的升级函数 |
| `arbitrary-send-eth` | 任意地址可接收 ETH |
| `tx-origin` | 使用 tx.origin 鉴权 |
| `unchecked-transfer` | 未检查 ERC-20 transfer 返回值 |

### 3.2 Aderyn

```bash
# 安装
cargo install aderyn

# 扫描
aderyn .
# 输出 report.md（Markdown 格式，更易读）
```

---

## 4. 形式化验证

```bash
# Certora Prover（功能最强）
certoraRun src/Vault.sol --verify Vault:specs/Vault.spec

# Halmos（基于 Foundry 的符号执行）
pip install halmos
halmos --contract VaultTest --function test_Withdraw
```

形式化验证适合：核心 DeFi 逻辑、关键安全属性（不变式）、高价值合约。

---

## 5. 测试网部署流程

```bash
# 1. 编译并检查大小
forge build
forge inspect MyContract deployedBytecode | wc -c  # 需 < 24576 字节

# 2. 估算 gas
forge script script/Deploy.s.sol --rpc-url $SEPOLIA_RPC --estimate-gas

# 3. 部署到测试网
forge script script/Deploy.s.sol \
  --rpc-url $SEPOLIA_RPC \
  --private-key $PRIVATE_KEY \
  --broadcast

# 4. 验证源码（Etherscan）
forge verify-contract $CONTRACT_ADDRESS MyContract \
  --etherscan-api-key $ETHERSCAN_KEY \
  --chain sepolia
```

---

## 6. 审计准备清单

提交审计前应完成：

- [ ] 所有单元测试通过（覆盖正常路径）
- [ ] Fuzz 测试（至少 10,000 runs）
- [ ] Invariant 测试（核心状态不变式）
- [ ] Slither / Aderyn 扫描并修复高危项
- [ ] 代码有完整 NatSpec 注释
- [ ] 设计文档（架构、流程图、安全考量）
- [ ] 已知风险与 trade-off 说明文档

---

## 参考资源

- [Foundry Testing 指南](https://book.getfoundry.sh/forge/tests)
- [Slither](https://github.com/crytic/slither)
- [Certora Prover](https://www.certora.com/)
- [Halmos](https://github.com/a16z/halmos)
- [Cyfrin Audits（开源报告）](https://github.com/Cyfrin/cyfrin-audit-reports)
