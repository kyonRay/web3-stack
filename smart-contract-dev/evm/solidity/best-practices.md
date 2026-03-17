# Solidity 安全模式与最佳实践

> **标签：** solidity、安全、CEI、重入、访问控制、审计

合约部署后不可变且公开，攻击者可任意调用。本文是编写安全 Solidity 代码的核心参考。

---

## 1. Checks-Effects-Interactions（CEI）模式

**防止重入攻击最重要的模式**：先完成所有状态变更（Checks + Effects），再做外部调用（Interactions）。

```solidity
// 不安全：先外部调用再改状态
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);           // Check
    (bool success, ) = msg.sender.call{value: amount}(""); // Interaction ← 危险！
    require(success);
    balances[msg.sender] -= amount;                    // Effect ← 太晚了
}

// 安全：CEI 顺序
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);           // Check
    balances[msg.sender] -= amount;                    // Effect（先改状态）
    (bool success, ) = msg.sender.call{value: amount}(""); // Interaction
    require(success);
}
```

多次外部调用时使用 OpenZeppelin `ReentrancyGuard`：

```solidity
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    function withdraw(uint256 amount) external nonReentrant {
        // ...
    }
}
```

---

## 2. 访问控制

所有特权函数必须做权限限制：

```solidity
// 方案 1：Ownable（单所有者）
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyContract is Ownable {
    function adminFunction() external onlyOwner {
        // ...
    }
}

// 方案 2：AccessControl（基于角色）
import "@openzeppelin/contracts/access/AccessControl.sol";

contract MyContract is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        // ...
    }
}
```

### 两步所有权转移（推荐）

```solidity
// Ownable2Step：先提名，再接受，防止误转到错误地址
import "@openzeppelin/contracts/access/Ownable2Step.sol";

contract MyContract is Ownable2Step {
    // 转移所有权需要两步：
    // 1. transferOwnership(newOwner) — 设置 pendingOwner
    // 2. acceptOwnership() — 由 pendingOwner 调用确认
}
```

---

## 3. 整数精度

```solidity
// Solidity 0.8+ 默认溢出检查
uint256 a = type(uint256).max;
uint256 b = a + 1; // 自动 revert

// 只在数学上可证明安全时使用 unchecked（省 gas）
for (uint256 i = 0; i < 10;) {
    // ...操作...
    unchecked { ++i; }  // i < 10 且 i 是 uint256，不会溢出
}

// 除法截断（Solidity 无浮点数）
uint256 result = 7 / 2; // = 3（向下截断）
// 需要精度时先乘后除
uint256 precise = (7 * 1e18) / 2; // = 3.5e18（用 1e18 作精度因子）
```

---

## 4. 避免常见陷阱

### 4.1 不用 tx.origin 做鉴权

```solidity
// 危险：tx.origin 是交易发起者，可被钓鱼合约利用
require(tx.origin == owner); // 不安全

// 安全：msg.sender 是直接调用者
require(msg.sender == owner);
```

### 4.2 循环中勿依赖 msg.value

```solidity
// 危险：msg.value 在整个交易中固定，但状态变化了
for (uint i = 0; i < recipients.length; i++) {
    balances[recipients[i]] += msg.value; // 每次都加同一个值！
}

// 正确：计算每人分配额
uint256 share = msg.value / recipients.length;
for (uint i = 0; i < recipients.length; i++) {
    balances[recipients[i]] += share;
}
```

### 4.3 可见性与 fallback

```solidity
// 明确所有函数可见性
function internal_helper() internal { }  // 只有本合约与子合约
function private_helper() private { }    // 只有本合约
function external_view() external view { }  // 外部调用，读状态

// fallback 与 receive 的区别
receive() external payable { }          // 仅接收 ETH（无 calldata）
fallback() external payable { }         // 所有未匹配调用（有 calldata）
```

### 4.4 避免 DoS 漏洞

```solidity
// 危险：遍历无限增长的数组
function distributeRewards() external {
    for (uint i = 0; i < users.length; i++) { // Gas 耗尽风险！
        _sendReward(users[i]);
    }
}

// 安全：分页处理 或 pull-over-push 模式
mapping(address => uint256) public pendingRewards;

function claimReward() external {
    uint256 amount = pendingRewards[msg.sender];
    pendingRewards[msg.sender] = 0;
    _send(msg.sender, amount);
}
```

---

## 5. 可升级合约安全

```solidity
// UUPS 代理（推荐）
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract MyContractV1 is UUPSUpgradeable, OwnableUpgradeable {
    uint256 public value;

    function initialize(uint256 _value) public initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
        value = _value;
    }

    function _authorizeUpgrade(address) internal override onlyOwner {}
}
```

**存储布局规则**：
- 升级时只在末尾追加新变量
- 不可插入、重排或删除已有变量
- 使用 `@openzeppelin/hardhat-upgrades` 检查存储布局兼容性

---

## 6. 错误处理最佳实践

```solidity
// 优先用 custom errors（省 gas + 便于调试）
error InsufficientBalance(address user, uint256 available, uint256 required);
error Unauthorized(address caller);

function withdraw(uint256 amount) external {
    if (balances[msg.sender] < amount) {
        revert InsufficientBalance(msg.sender, balances[msg.sender], amount);
    }
    // ...
}
```

---

## 7. 事件与 Indexed 字段

```solidity
// 对所有状态变更发出事件
event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);

// indexed 字段可被链下过滤（最多 3 个 indexed）
emit Transfer(msg.sender, recipient, amount);

// 避免在 indexed 字段放大数据（动态类型 indexed 会被 keccak256 哈希）
event WithData(bytes indexed data); // data 被哈希，无法还原！
```

---

## 8. 部署前安全清单

- [ ] 所有外部调用遵循 CEI 或使用 `nonReentrant`
- [ ] 特权函数均有访问控制
- [ ] 不使用 `tx.origin` 做鉴权
- [ ] 无硬编码地址（用构造参数或治理）
- [ ] 升级合约的存储布局已文档化且工具验证
- [ ] 合约体积 < 24 KB（可用 Diamond 模式分片）
- [ ] 测试覆盖正常路径、边界与攻击场景
- [ ] 高价值合约经过第三方专业审计
- [ ] 已有暂停机制或紧急出口

---

## 参考资源

- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [SWC Registry（漏洞分类）](https://swcregistry.io/)
- [Consensys Smart Contract Best Practices](https://consensys.github.io/smart-contract-best-practices/)
- [Solodit（审计报告搜索）](https://solodit.xyz/)
