# Vyper 与其他 EVM 语言

> **标签：** vyper、yul、fe、cairo、evm 语言

除 Solidity 外，EVM 生态还有多种合约语言，各有不同的安全哲学与适用场景。

---

## 1. Vyper

Vyper 是 Python 风格的合约语言，以**安全性**和**可审计性**为首要目标：

### 核心设计原则

- **无继承**：消除复杂的继承链，合约逻辑一目了然
- **无递归**：防止栈溢出与 gas 不可预期问题
- **无内联汇编**（默认）：减少低级漏洞机会
- **强类型**：无隐式类型转换
- **Bounds checking**：数组越界自动检查

```python
# Vyper ERC-20 示例（部分）
from ethereum.ercs import IERC20

implements: IERC20

balances: HashMap[address, uint256]
allowances: HashMap[address, HashMap[address, uint256]]
totalSupply: public(uint256)

@external
def transfer(_to: address, _value: uint256) -> bool:
    assert self.balances[msg.sender] >= _value
    self.balances[msg.sender] -= _value
    self.balances[_to] += _value
    log Transfer(msg.sender, _to, _value)
    return True
```

### 使用场景

- DeFi 核心合约（Curve、Yearn 的部分合约用 Vyper）
- 安全审计更简单的场景
- 团队偏好 Python 风格语法

### 参考资源

- [Vyper 官方文档](https://docs.vyperlang.org/)
- [Titanoboa（Vyper 测试框架）](https://github.com/vyperlang/titanoboa)

---

## 2. Yul / Yul+

Yul 是 Solidity 的中间语言，也可直接编写（内联汇编或独立文件）：

```javascript
// Yul 独立合约：极简 ERC-20 transfer
object "Token" {
    code {
        // 构造函数
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }
    object "runtime" {
        code {
            let sig := shr(224, calldataload(0))
            switch sig
            case 0xa9059cbb { // transfer(address,uint256)
                let to := calldataload(4)
                let amount := calldataload(36)
                // 读取余额 slot: keccak256(from ++ balanceSlot)
                mstore(0x00, caller())
                mstore(0x20, 0)  // balances slot = 0
                let fromSlot := keccak256(0x00, 0x40)
                let balance := sload(fromSlot)
                require(iszero(lt(balance, amount)))
                sstore(fromSlot, sub(balance, amount))
                // 写入接收方余额
                mstore(0x00, to)
                let toSlot := keccak256(0x00, 0x40)
                sstore(toSlot, add(sload(toSlot), amount))
                mstore(0x00, 1)
                return(0x00, 0x20)
            }
        }
    }
}
```

**适合场景**：
- 极致 gas 优化（如 Uniswap V3 的 tick bitmap 操作）
- 实现特殊 EVM 操作（创建代理合约、特殊 slot 操作）
- 理解 EVM 底层机制

---

## 3. Fe

Fe 是 Rust 风格的 EVM 合约语言，目前仍在开发中：

```rust
// Fe 语法示例
contract Counter {
    count: u256

    pub fn increment(mut self) {
        self.count += 1
    }

    pub fn get(self) -> u256 {
        return self.count
    }
}
```

**特点**：
- 类 Rust 语法，静态类型
- 避免 Solidity 的一些历史遗留问题
- 目前不推荐生产使用（仍在 alpha 阶段）

---

## 4. Cairo（StarkNet）

Cairo 是 StarkNet 的原生语言，专为 ZK 证明设计：

```rust
// Cairo 合约（Starknet）
#[starknet::contract]
mod ERC20 {
    use starknet::ContractAddress;

    #[storage]
    struct Storage {
        balances: LegacyMap::<ContractAddress, u256>,
        total_supply: u256,
    }

    #[abi(embed_v0)]
    impl ERC20Impl of super::IERC20<ContractState> {
        fn transfer(ref self: ContractState, recipient: ContractAddress, amount: u256) -> bool {
            let sender = starknet::get_caller_address();
            self._transfer(sender, recipient, amount);
            true
        }
    }
}
```

**特点**：
- 非 EVM 兼容（StarkNet 专用）
- 证明友好：所有操作可被 STARK 证明
- 类 Rust 语法，强类型系统

---

## 5. 语言选型建议

| 语言 | 适用场景 | 成熟度 |
|------|---------|--------|
| **Solidity** | 通用 EVM 合约（首选） | ⭐⭐⭐⭐⭐ |
| **Vyper** | 安全优先的 DeFi 核心合约 | ⭐⭐⭐⭐ |
| **Yul** | Gas 极致优化的核心函数 | ⭐⭐⭐ |
| **Cairo** | StarkNet 合约（专用） | ⭐⭐⭐⭐ |
| **Fe** | 实验/研究（不推荐生产） | ⭐⭐ |
