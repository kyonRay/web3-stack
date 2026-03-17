# Gas 优化

> **标签：** solidity、gas、优化、evm、性能、存储

EVM 执行每步操作都有 gas 成本。Gas 优化能降低用户成本、提升协议竞争力。本文从 EVM 成本模型出发，系统梳理最有效的优化手段。

---

## 1. EVM 成本心智模型

优化前必须了解相对成本（约数）：

| 操作 | Gas（约） | 备注 |
|------|-----------|------|
| SSTORE（新非零值） | 20,000 | 最贵，写新存储槽 |
| SSTORE（非零→非零） | 2,900 | 修改已存在的值 |
| SLOAD（冷） | 2,100 | 首次访问该 slot |
| SLOAD（热）| 100 | 同交易内已访问过 |
| CALL（含 ETH，冷） | 9,000 + 2,600 | 外部调用 |
| LOG0–LOG4 | 375 + 375/主题 | 发出事件 |
| KECCAK256 | 30 + 6/字 | 哈希运算 |
| ADD/MUL | 3–5 | 便宜 |
| MLOAD/MSTORE | 3 | 内存读写 |

**核心原则：最小化存储读写（SSTORE/SLOAD），用计算换存储。**

---

## 2. 存储槽打包（Slot Packing）

EVM 按 32 字节 slot 存状态，连续声明的小变量可共享一个 slot：

```solidity
// 未优化：占 3 个 slot
contract Unoptimized {
    uint256 a;    // slot 0 (32 bytes)
    uint128 b;    // slot 1 (16 bytes used)
    uint128 c;    // slot 2 (16 bytes used)
}

// 优化：b 和 c 打包到同一 slot
contract Optimized {
    uint256 a;    // slot 0
    uint128 b;    // slot 1（低 128 bits）
    uint128 c;    // slot 1（高 128 bits）
    // 节省一个 SSTORE/SLOAD！
}
```

**注意**：打包仅在同一交易中同时读写多个打包变量时有效。若单独访问，掩码开销可能抵消优势。

---

## 3. 缓存存储变量到内存

```solidity
// 未优化：每次迭代 2x SLOAD
function sumArray(uint256[] memory arr) external view returns (uint256) {
    uint256 total;
    for (uint i = 0; i < arr.length; i++) {
        total += arr[i];
        if (total > maxTotal) break; // maxTotal 每次都 SLOAD
    }
    return total;
}

// 优化：缓存到内存变量
function sumArray(uint256[] memory arr) external view returns (uint256) {
    uint256 total;
    uint256 _maxTotal = maxTotal; // 一次 SLOAD，后续用内存
    for (uint i = 0; i < arr.length; i++) {
        total += arr[i];
        if (total > _maxTotal) break; // MLOAD（便宜）
    }
    return total;
}
```

---

## 4. 避免在独立 slot 中使用小整数

```solidity
// 较贵：uint8 在独立 slot 中比 uint256 更贵（需要掩码操作）
uint8 public flag;  // slot 0（但 EVM 仍按 32 字节操作）

// 更好：独立 slot 用 uint256
uint256 public flag;  // 同样 1 个 slot，但操作更直接

// uint8 适合打包场景
uint128 public a;
uint64 public b;
uint32 public c;
uint32 public d;  // a+b+c+d = 256 bits，打包到 1 个 slot
```

---

## 5. calldata 替代 memory

```solidity
// memory：将 calldata 复制到内存（2x 成本）
function process(uint256[] memory ids) external {
    // ...
}

// calldata：直接读 calldata，不复制
function process(uint256[] calldata ids) external {
    // 节省 CALLDATACOPY gas
}
```

---

## 6. unchecked 运算

```solidity
// 0.8+ 默认溢出检查（每次运算额外约 30 gas）
for (uint256 i = 0; i < 100; i++) { }  // 100 次 overflow check

// unchecked：仅在数学上可证明安全时使用
for (uint256 i = 0; i < 100;) {
    // 循环体
    unchecked { ++i; }  // 无溢出检查，++i 略优于 i++
}
```

---

## 7. Custom Errors 替代 require 字符串

```solidity
// require + string：在 calldata 中携带字符串（部署和调用都贵）
require(amount > 0, "Amount must be positive");

// custom error：只需 4 字节选择器 + 参数（便宜很多）
error InvalidAmount(uint256 amount);
if (amount == 0) revert InvalidAmount(amount);
```

---

## 8. 位图替代 bool mapping

```solidity
// mapping(uint256 => bool)：每个 bool 占一个完整 slot（20,000 gas/bit）
mapping(uint256 => bool) public claimed;

// 位图：1 个 uint256 存 256 个 bool
mapping(uint256 => uint256) private claimedBitmap;

function isClaimed(uint256 tokenId) public view returns (bool) {
    uint256 word = tokenId / 256;
    uint256 bit = tokenId % 256;
    return (claimedBitmap[word] >> bit) & 1 == 1;
}

function setClaimed(uint256 tokenId) internal {
    uint256 word = tokenId / 256;
    uint256 bit = tokenId % 256;
    claimedBitmap[word] |= (1 << bit);
}
```

ERC-721A 等项目采用此方案将 NFT mint 成本降低 5-10x。

---

## 9. 链上少存、用事件记历史

```solidity
// 昂贵：存储历史记录
mapping(uint256 => Transaction[]) public history; // 每条约 20,000 gas

// 便宜：用事件（约 375 + 250/32bytes gas）
event TransactionRecorded(
    address indexed from,
    address indexed to,
    uint256 amount,
    uint256 timestamp
);
// 链下索引消费事件（The Graph / Dune）
```

---

## 10. 编译器优化

```json
// foundry.toml
[profile.default]
optimizer = true
optimizer_runs = 200   // 常规合约
# optimizer_runs = 1   // 只部署一次、不常调用的合约（如工厂）

[profile.deploy]
optimizer_runs = 1     // 降低部署成本

# 启用 viaIR（更激进优化，编译更慢）
via_ir = true
```

```
optimizer_runs 含义：
  高（10,000+）：优化调用成本，增加部署成本
  低（1–200）：优化部署成本，调用略贵
  适合场景：DEX 路由器用高值；工厂合约用低值
```

---

## 11. 短路与条件排序

```solidity
// 把便宜/更可能失败的条件放前面
require(amount > 0 && token != address(0) && balances[msg.sender] >= amount);
// ↑ amount > 0 最便宜，先检查；balances SLOAD 最贵，放最后

// 短路避免不必要的 SLOAD
if (paused || balances[msg.sender] < MIN) revert();
// paused 是布尔（热 slot），先检查
```

---

## Gas 优化工具

| 工具 | 用途 |
|------|------|
| `forge test --gas-report` | 函数级 gas 报告 |
| `forge snapshot` | 跨 commit 对比 gas 变化 |
| `gasol` | Solidity 优化建议 |
| `hardhat-gas-reporter` | Hardhat gas 报告 |

---

## 参考资源

- [EVM Opcodes Gas 表](https://www.evm.codes/)
- [EIP-2929（柏林升级冷热 SLOAD）](https://eips.ethereum.org/EIPS/eip-2929)
- [ERC-721A](https://www.erc721a.org/)（位图 mint 优化）
- [Foundry Gas Snapshots](https://book.getfoundry.sh/forge/gas-snapshots)
