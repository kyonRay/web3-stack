# Solidity 语言深入

> **标签：** solidity、类型系统、存储模型、abi、继承、汇编

本文深入 Solidity 语言核心特性，面向已有基本语法基础、希望深入理解底层机制的开发者。

---

## 1. 类型系统

### 1.1 值类型（Value Types）

| 类型 | 说明 | 默认值 |
|------|------|--------|
| `bool` | 布尔 | `false` |
| `uint8`–`uint256` | 无符号整数（步长 8 bit） | `0` |
| `int8`–`int256` | 有符号整数 | `0` |
| `address` | 20 字节地址 | `address(0)` |
| `address payable` | 可接收 ETH 的地址 | — |
| `bytes1`–`bytes32` | 固定字节数组 | 零字节 |
| `enum` | 枚举（内部为 uint8 起） | 第一个值 |

值类型在赋值时**复制**，互不影响。

### 1.2 引用类型（Reference Types）

| 类型 | 数据位置 | 说明 |
|------|---------|------|
| `bytes` | storage/memory | 动态字节数组 |
| `string` | storage/memory | UTF-8 字符串 |
| `T[]` / `T[N]` | storage/memory | 动态/固定数组 |
| `mapping(K => V)` | storage only | 哈希映射 |
| `struct` | storage/memory | 用户定义结构 |

引用类型必须明确指定**数据位置**：`storage`、`memory` 或 `calldata`。

### 1.3 函数类型

```solidity
// 函数类型变量
function(uint256) external returns (bool) public callback;

// 内部函数类型
function(uint256, uint256) internal pure returns (uint256) op;
op = add;
op = sub;

function add(uint256 a, uint256 b) internal pure returns (uint256) {
    return a + b;
}
```

---

## 2. 数据位置与 Gas

### 存储层级（Gas 成本从高到低）

```
storage（持久化）: 20,000 gas（新写）/ 2,100 gas（冷读）/ 100 gas（热读）
memory（函数内临时）: 随大小按 gas 收费，初始便宜
calldata（只读外部输入）: 最便宜，不可修改
stack（EVM 栈）: 最多 1024 个 256-bit 值，免费但容量有限
```

### 关键规则

```solidity
// storage 引用（不复制，修改会持久化）
MyStruct storage s = structs[id];
s.value = 100; // 直接修改 storage

// memory 复制（不影响 storage）
MyStruct memory m = structs[id];
m.value = 100; // 只改了内存副本

// calldata（最节省 gas 的只读参数）
function process(uint256[] calldata ids) external {
    // ids 直接读 calldata，无需复制到 memory
}
```

---

## 3. 存储布局

### 3.1 基本规则

- 每个状态变量占一个或多个 32 字节 **slot**
- slot 编号从 0 开始顺序分配
- 值类型小于 32 字节时可**打包**到同一 slot

```solidity
contract StorageLayout {
    uint256 a;    // slot 0
    uint128 b;    // slot 1（低 128 bits）
    uint128 c;    // slot 1（高 128 bits）—— 与 b 打包
    address d;    // slot 2（20 bytes）
    uint8 e;      // slot 2（21st byte）—— 与 d 打包
    bool f;       // slot 2（22nd byte）
}
```

### 3.2 动态类型的 slot

```
mapping(key => value): slot 本身不存数据
  实际 slot = keccak256(abi.encode(key, mappingSlot))

动态数组: slot 存长度
  第 i 个元素的 slot = keccak256(arraySlot) + i

bytes/string（短字节串 < 32 bytes）:
  存在同一 slot（最低位存 length * 2）
  长度 >= 32 时：slot 存 length * 2 + 1，数据从 keccak256(slot) 起
```

### 3.3 升级合约的存储风险

```solidity
// V1
contract V1 {
    uint256 public a;  // slot 0
    uint256 public b;  // slot 1
}

// V2 错误：在 a 和 b 之间插入 c，导致 b 被覆盖
contract V2 {
    uint256 public a;  // slot 0
    uint256 public c;  // slot 1 ← 错！b 的数据在这里
    uint256 public b;  // slot 2
}

// V2 正确：只在末尾追加
contract V2Correct {
    uint256 public a;  // slot 0
    uint256 public b;  // slot 1
    uint256 public c;  // slot 2（新增）
}
```

---

## 4. ABI 编码

### 4.1 函数选择器（Selector）

函数选择器是函数签名 keccak256 的前 4 字节：

```solidity
// transfer(address,uint256) 的 selector
bytes4 selector = bytes4(keccak256("transfer(address,uint256)"));
// = 0xa9059cbb

// 等同于
bytes4 selector = IERC20.transfer.selector;
```

### 4.2 编码方式

```solidity
// abi.encode：ABI 标准编码（带填充）
bytes memory encoded = abi.encode(uint256(1), address(0x123));
// 每个参数填充到 32 字节

// abi.encodePacked：紧密打包（无填充，不安全用于哈希）
bytes memory packed = abi.encodePacked(address(0x123), uint256(1));
// 注意：abi.encodePacked(a, b) == abi.encodePacked(c, d) 可能碰撞

// abi.encodeWithSelector：拼接函数选择器
bytes memory calldata = abi.encodeWithSelector(
    IERC20.transfer.selector,
    recipient,
    amount
);

// abi.decode：解码
(uint256 a, address b) = abi.decode(data, (uint256, address));
```

---

## 5. 继承与接口

### 5.1 多重继承（C3 线性化）

```solidity
contract A {
    function foo() public virtual returns (string memory) { return "A"; }
}
contract B is A {
    function foo() public virtual override returns (string memory) { return "B"; }
}
contract C is A {
    function foo() public virtual override returns (string memory) { return "C"; }
}
// MRO: D → B → C → A
contract D is B, C {
    function foo() public override(B, C) returns (string memory) {
        return super.foo(); // 调用 C.foo()（最右继承链）
    }
}
```

### 5.2 接口 vs 抽象合约

```solidity
// 接口：只声明，不实现
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
}

// 抽象合约：部分实现
abstract contract Ownable {
    address public owner;
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    // 留给子合约实现
    function renounceOwnership() public virtual;
}
```

---

## 6. 内联汇编（Assembly）

Yul 汇编允许直接操作 EVM，用于极致优化或实现特殊功能：

```solidity
function efficientHash(bytes32 a, bytes32 b) internal pure returns (bytes32 result) {
    assembly {
        mstore(0x00, a)
        mstore(0x20, b)
        result := keccak256(0x00, 0x40)
    }
}

// 读取任意存储 slot
function readSlot(uint256 slot) internal view returns (bytes32 value) {
    assembly {
        value := sload(slot)
    }
}
```

**注意**：汇编绕过了 Solidity 的安全检查，使用前确保完全理解操作含义。

---

## 7. 错误处理

```solidity
// Custom Error（推荐）— 比 require string 省 gas
error InsufficientBalance(uint256 available, uint256 required);
if (balance < amount) revert InsufficientBalance(balance, amount);

// require + string（兼容旧代码）
require(amount > 0, "Amount must be positive");

// assert（检查内部不变式，Gas 全耗尽）
assert(totalSupply == sumOfBalances); // 只用于内部一致性检查

// try/catch（外部调用错误处理）
try externalContract.call() returns (uint256 result) {
    // 成功
} catch Error(string memory reason) {
    // require 失败
} catch (bytes memory lowLevelData) {
    // 其他失败
}
```

---

## 参考资源

- [Solidity 文档 — 类型](https://docs.soliditylang.org/en/latest/types.html)
- [Solidity 文档 — 存储布局](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)
- [ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html)
- [EVM Codes（操作码参考）](https://www.evm.codes/)
