# EVM 内部原理

> **标签：** evm、字节码、操作码、状态树、mpt、内存模型

深入理解 EVM 内部机制，有助于编写更优化的合约、理解反编译结果、进行链开发工作。

---

## 1. EVM 执行模型

EVM 是**基于栈的 256-bit 虚拟机**：

```
执行上下文
├── Code（合约字节码，只读）
├── Stack（最大 1024 个 256-bit 元素）
├── Memory（按需扩展，字节寻址）
├── Storage（持久化 key-value，32字节→32字节）
├── Calldata（调用时传入，只读）
├── ReturnData（上一次调用的返回值）
└── 环境变量（address、caller、value、gasPrice 等）
```

---

## 2. 字节码结构

```
合约字节码分为两部分：
1. 创建代码（Constructor bytecode）：只在部署时执行，末尾 RETURN 运行时代码
2. 运行时代码（Runtime bytecode）：部署后存储在链上，每次调用执行
```

### 函数调度器（Dispatcher）

Solidity 编译的运行时代码开头通常是函数选择器分发：

```assembly
PUSH1 0x00
CALLDATALOAD     ; 加载 calldata 前 32 字节
PUSH1 0xe0
SHR              ; 右移 224 位，取前 4 字节（函数选择器）
DUP1
PUSH4 0xa9059cbb ; transfer(address,uint256) 的选择器
EQ
PUSH1 0x40       ; 跳转到 transfer 函数入口
JUMPI            ; 若相等则跳转
```

---

## 3. 内存布局

EVM 内存是字节寻址的线性空间，按 32 字节对齐分配：

```
0x00 - 0x3f  : 临时哈希/加密计算用（Solidity 规范预留）
0x40 - 0x5f  : 空闲内存指针（Free Memory Pointer，初始 = 0x80）
0x60 - 0x7f  : 零值槽（用于清零等操作）
0x80+        : 实际分配的内存开始
```

```solidity
assembly {
    let freeMemPtr := mload(0x40)  // 读取空闲内存指针
    mstore(freeMemPtr, value)       // 存值到空闲内存
    mstore(0x40, add(freeMemPtr, 0x20)) // 更新空闲内存指针
}
```

---

## 4. Storage 布局与 MPT

以太坊状态存储在 **Merkle Patricia Trie（MPT）**：

```
全局状态 MPT
  key   = keccak256(account_address)
  value = RLP(nonce, balance, storageRoot, codeHash)

合约存储 MPT（每个合约独立）
  key   = keccak256(slot_index)（对于动态类型）
  value = slot_value
```

### Storage Proof（状态证明）

```bash
# 获取账户存储证明
cast proof $CONTRACT_ADDRESS $SLOT_INDEX --rpc-url $RPC_URL

# 返回：账户证明路径 + 存储槽证明路径（Merkle 路径节点列表）
# 任何人可用 stateRoot 独立验证
```

---

## 5. ABI 编码深入

### 静态类型编码

```
abi.encode(uint256(1), address(0xABC), bool(true))
→
0000000000000000000000000000000000000000000000000000000000000001  (uint256 1)
000000000000000000000000000000000000000000000000000000000000_0abc  (address，高位补零)
0000000000000000000000000000000000000000000000000000000000000001  (bool，补零)
```

### 动态类型编码（头尾编码）

```
abi.encode(uint256(1), bytes("hello"))
→
0000000000000000000000000000000000000000000000000000000000000001  (uint256 1)
0000000000000000000000000000000000000000000000000000000000000040  (bytes 的偏移量 = 64 bytes)
0000000000000000000000000000000000000000000000000000000000000005  (bytes 长度 = 5)
68656c6c6f000000000000000000000000000000000000000000000000000000  ("hello" 右边补零)
```

---

## 6. Gas 计量机制

### 冷热存储（EIP-2929）

```
首次访问地址/slot（冷）：
  SLOAD  cold = 2100 gas
  CALL   cold = 2600 gas

同一交易内再次访问（热）：
  SLOAD  hot = 100 gas
  CALL   hot = 100 gas
```

`access_list`（EIP-2930）允许预热地址和存储 slot，降低实际执行时成本（适合已知会访问的地址）。

### Memory 扩展成本

内存按 32 字节（word）扩展，成本公式：

```
memory_cost = 3 * memory_size_words + memory_size_words^2 / 512
```

超过约 32 KB 时成本开始快速上升（防止无限内存使用）。

---

## 7. DELEGATECALL 存储冲突

```
ProxyContract（地址 0xA）存储：
  slot 0: implementation_address = 0xB

ImplementationContract（地址 0xB）的代码：
  slot 0: owner
  slot 1: balance

问题：Proxy 通过 DELEGATECALL 执行 Impl 代码时：
  slot 0 = 0xA 的 slot 0 = 当前 implementation 地址（非 owner！）
  Impl 写 owner 实际上覆盖了 Proxy 的 implementation 地址！

解决：EIP-1967 使用固定的远端 slot（keccak256("eip1967.proxy.implementation") - 1）
```

---

## 8. 字节码反编译工具

| 工具 | 说明 |
|------|------|
| [Dedaub](https://app.dedaub.com/) | 在线反编译，识别函数和变量 |
| [Panoramix](https://github.com/palkeo/panoramix) | 反汇编到伪代码 |
| [heimdall-rs](https://github.com/Jon-Becker/heimdall-rs) | Rust 实现的反编译 + 反汇编 |
| `cast disassemble` | Foundry 字节码反汇编 |

```bash
# Foundry 反汇编
cast disassemble $(cast code $CONTRACT_ADDRESS --rpc-url $RPC_URL)
```

---

## 参考资源

- [EVM Opcodes 参考](https://www.evm.codes/)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [EIP-2929（冷热访问）](https://eips.ethereum.org/EIPS/eip-2929)
- [ABI 规范](https://docs.soliditylang.org/en/latest/abi-spec.html)
