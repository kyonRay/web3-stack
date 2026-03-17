# 开发框架与工具

> **标签：** foundry、hardhat、openzeppelin、remix、测试框架

智能合约开发常用 **Foundry** 或 **Hardhat** 做编译、测试、脚本与部署；**OpenZeppelin** 提供生产级标准合约库。

---

## 1. Foundry（推荐）

Foundry 是目前最流行的 Solidity 开发框架，使用 Rust 编写，速度极快。

### 核心工具

| 工具 | 说明 |
|------|------|
| `forge` | 编译、测试、部署 |
| `cast` | 命令行与链交互（读合约、发交易） |
| `anvil` | 本地节点（支持 fork 主网） |
| `chisel` | 交互式 Solidity REPL |

### 安装与基本用法

```bash
# 安装
curl -L https://foundry.paradigm.xyz | bash
foundryup

# 新建项目
forge init my-project
cd my-project

# 编译
forge build

# 测试
forge test
forge test -vvv           # 详细输出
forge test --gas-report   # gas 报告
forge test --fuzz-runs 1000  # 模糊测试

# 部署（脚本方式）
forge script script/Deploy.s.sol \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify --etherscan-api-key $ETHERSCAN_KEY
```

### 测试示例

```solidity
// test/Counter.t.sol
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter counter;
    address alice = makeAddr("alice");

    function setUp() public {
        counter = new Counter();
        vm.deal(alice, 10 ether);
    }

    function test_Increment() public {
        counter.increment();
        assertEq(counter.count(), 1);
    }

    // 模糊测试：Foundry 自动生成随机 amount
    function testFuzz_Add(uint256 amount) public {
        vm.assume(amount < 1000); // 过滤无效输入
        counter.add(amount);
        assertEq(counter.count(), amount);
    }

    // 测试 revert
    function test_RevertIfNotOwner() public {
        vm.prank(alice); // 以 alice 身份调用
        vm.expectRevert("Not owner");
        counter.adminReset();
    }

    // fork 测试（模拟主网状态）
    function test_ForkUniswap() public {
        vm.createSelectFork("mainnet"); // foundry.toml 配置 RPC
        // 与真实的 Uniswap 交互
    }
}
```

### foundry.toml 配置

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
optimizer = true
optimizer_runs = 200
via_ir = false

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
arbitrum = "${ARBITRUM_RPC_URL}"

[etherscan]
mainnet = { key = "${ETHERSCAN_KEY}" }
arbitrum = { key = "${ARBISCAN_KEY}", url = "https://api.arbiscan.io/api" }
```

---

## 2. Hardhat

Hardhat 是 Node.js 生态的合约开发框架，插件丰富，适合全栈团队。

```bash
npm install --save-dev hardhat
npx hardhat init

# 编译
npx hardhat compile

# 测试（使用 Ethers.js）
npx hardhat test

# 部署脚本
npx hardhat run scripts/deploy.ts --network sepolia
```

### 常用插件

| 插件 | 功能 |
|------|------|
| `@nomicfoundation/hardhat-toolbox` | 集合包：ethers、verify、gas-reporter |
| `@openzeppelin/hardhat-upgrades` | 可升级合约部署与验证 |
| `hardhat-contract-sizer` | 合约大小统计 |
| `@nomiclabs/hardhat-etherscan` | Etherscan 验证 |

### Hardhat vs Foundry 选型

| 维度 | Foundry | Hardhat |
|------|---------|---------|
| 测试语言 | Solidity | JavaScript/TypeScript |
| 速度 | 极快（Rust） | 较慢（Node.js） |
| 模糊测试 | 内置 | 需插件 |
| 插件生态 | 较少 | 丰富 |
| 全栈集成 | 一般 | 优秀 |
| 推荐场景 | 合约重度开发 | 全栈 DApp 项目 |

---

## 3. OpenZeppelin Contracts

生产环境的合约标准库，经过大量审计，强烈建议优先使用：

```bash
forge install OpenZeppelin/openzeppelin-contracts
# 或
npm install @openzeppelin/contracts
```

### 常用模块

```solidity
// 代币
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";

// 访问控制
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/access/Ownable2Step.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

// 安全
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

// 可升级（upgradeable 版本）
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

// 治理
import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";
```

---

## 4. Remix IDE

浏览器端 IDE，适合快速验证和教学：

- [remix.ethereum.org](https://remix.ethereum.org)
- 支持直连 MetaMask、测试网、本地 Hardhat/Anvil
- 内置调试器，可单步追踪 EVM 执行
- 适合快速实验，不适合大型项目

---

## 5. 安全工具

### 5.1 Slither（静态分析）

```bash
# 安装
pip install slither-analyzer

# 分析整个项目
slither .

# 常见输出类型
# - reentrancy-eth：重入漏洞
# - arbitrary-send-eth：任意 ETH 转账
# - uninitialized-local：未初始化变量
# - shadowing-local：变量遮蔽
```

### 5.2 Mythril（符号执行）

```bash
# 安装
pip install mythril

# 分析单个合约
myth analyze src/MyContract.sol --solc-json foundry.json
```

### 5.3 Echidna（模糊测试）

Echidna 是基于属性的智能合约模糊测试工具：

```solidity
// echidna 测试：属性应始终为真
contract EchidnaTest {
    uint256 balance;

    function deposit(uint256 amount) public {
        balance += amount;
    }

    // echidna_* 函数应始终返回 true
    function echidna_balance_never_negative() public view returns (bool) {
        return balance >= 0;
    }
}
```

### 5.4 安全工具对比

| 工具 | 类型 | 优势 | 适合阶段 |
|------|------|------|---------|
| **Slither** | 静态分析 | 快速，覆盖常见模式 | 开发阶段日常扫描 |
| **Mythril** | 符号执行 | 深度路径分析 | 审计前深度检查 |
| **Echidna** | 模糊测试 | 随机属性验证 | 核心逻辑测试 |
| **Foundry Fuzz** | 模糊测试 | 集成在开发流程中 | 日常测试 |
| **Certora Prover** | 形式化验证 | 数学级别证明 | 高价值合约审计 |

---

## 6. 推荐开发工作流

```
1. 设计合约接口（写接口文件 + NatSpec 注释）
2. 实现核心逻辑（使用 OpenZeppelin 基础合约）
3. 编写测试（Foundry：单元 + fuzz + invariant）
4. 运行 Slither 安全扫描
5. 本地 fork 主网集成测试
6. 部署到测试网（forge script --broadcast）
7. 验证源码（--verify --etherscan-api-key）
8. 测试网测试通过后主网部署
```

---

## 参考资源

- [Foundry Book](https://book.getfoundry.sh/)
- [Slither 文档](https://github.com/crytic/slither)
- [Echidna 文档](https://github.com/crytic/echidna)
- [Mythril 文档](https://github.com/ConsenSys/mythril)
