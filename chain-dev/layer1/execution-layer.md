# 执行层

> **标签：** evm、状态转换、区块执行、opcodes、执行层

执行层负责**执行交易、运行 EVM 字节码、维护全局状态**。是链开发者必须深入理解的核心模块。

---

## 1. 执行层在以太坊架构中的位置

The Merge 后，执行层与共识层分离：

- **执行层**：EVM 执行 → 状态转换 → 状态树更新 → 生成 receipts
- **共识层**：区块排序 → PoS 最终化 → 通知执行层 via Engine API

---

## 2. 以太坊状态机

以太坊是一个**确定性状态机**：

```
State(t) + Transactions → State(t+1)
```

全局状态是所有账户的映射：`Address → AccountState`

```
AccountState {
  nonce:       uint64        // 防重放，EOA 每发一笔交易 +1
  balance:     uint256       // ETH 余额（wei）
  codeHash:    bytes32       // 合约字节码的 keccak256，EOA 为空哈希
  storageRoot: bytes32       // 合约存储 MPT 的根哈希
}
```

---

## 3. 交易类型与处理流程

### 3.1 交易类型

| Type | 说明 |
|------|------|
| `0x00`（Legacy） | 旧格式，使用 `gasPrice` |
| `0x01`（EIP-2930） | 引入 access list |
| `0x02`（EIP-1559） | `maxFeePerGas` + `maxPriorityFeePerGas` |
| `0x03`（EIP-4844） | 带 blob sidecar，用于 L2 数据可用性 |

### 3.2 区块执行流程

```
1. 验证区块头（时间戳、gasLimit、baseFee 计算等）
2. 对区块内每笔交易：
   a. 验证签名与 nonce
   b. 扣除预付 gas（upfront cost）
   c. 执行 EVM 调用（或转账）
   d. 计算实际 gas 消耗，退还剩余
   e. 生成 receipt（logs、status、gasUsed）
3. 计算 stateRoot、receiptsRoot、logsBloom
4. 提交新 stateRoot 给共识客户端
```

---

## 4. EVM 执行模型

EVM 是**基于栈的虚拟机**：

- **栈**（Stack）：最大 1024 个 256-bit 元素，操作数从栈顶读写
- **内存**（Memory）：字节可寻址，按 32 字节对齐扩展，按 gas 收费
- **存储**（Storage）：256-bit key → 256-bit value，持久化，最贵
- **calldata**：调用时传入的只读字节，用于 ABI 编码的函数参数
- **returndata**：被调用合约的返回值

### 4.1 操作码分类

| 类别 | 示例 | 说明 |
|------|------|------|
| 算术 | `ADD` `MUL` `DIV` `MOD` | 基本运算（3-5 gas） |
| 比较/位操作 | `LT` `GT` `AND` `OR` `XOR` | 逻辑运算 |
| 环境 | `CALLER` `CALLVALUE` `BLOCKHASH` | 区块与调用上下文 |
| 存储 | `SLOAD`（冷 2100）`SSTORE`（新写 20000）| 最贵，优化重点 |
| 内存 | `MLOAD` `MSTORE` `MSIZE` | 内存读写（3 gas + 扩展费） |
| 跳转 | `JUMP` `JUMPI` `JUMPDEST` | 控制流 |
| 日志 | `LOG0`–`LOG4` | 发出事件，不影响状态 |
| 调用 | `CALL` `DELEGATECALL` `STATICCALL` `CREATE` | 外部调用，贵且复杂 |
| 返回 | `RETURN` `REVERT` `STOP` | 结束执行 |

### 4.2 CALL vs DELEGATECALL

```
CALL:         执行目标合约代码，在目标合约的存储上下文中
DELEGATECALL: 执行目标合约代码，在调用方合约的存储上下文中
              （代理合约的核心：逻辑合约 + 存储合约分离）
```

---

## 5. Gas 机制

- **baseFee**：由协议根据上块 gasUsed 动态调整（EIP-1559），被销毁
- **priorityFee（tip）**：用户设置，给验证者
- **gasLimit**：用户设置的上限；区块也有 gasLimit（目标 15M，最大 30M）
- **EIP-2929**（柏林升级）：冷热地址区分，首次访问更贵（防 DoS）
- **EIP-3651**（上海升级）：`COINBASE` 预热，降低 MEV 相关成本

---

## 6. Revm：可嵌入的 EVM 实现

[revm](https://github.com/bluealloy/revm) 是 Rust 实现的 EVM，被 Reth、Foundry/Anvil、Alloy 等广泛使用：

```rust
use revm::{Evm, primitives::*};

let mut evm = Evm::builder()
    .with_db(db)
    .modify_tx_env(|tx| {
        tx.caller = address!("...");
        tx.data = calldata;
    })
    .build();

let result = evm.transact_commit()?;
```

---

## 参考资源

- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [EVM Opcodes Reference](https://www.evm.codes/)
- [revm GitHub](https://github.com/bluealloy/revm)
- [EIP-1559 规范](https://eips.ethereum.org/EIPS/eip-1559)
