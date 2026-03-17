# ZK Rollup

> **标签：** zk-rollup、zkSync、polygon-zkevm、starknet、scroll、zk-evm

ZK Rollup 通过**有效性证明（Validity Proof）**在密码学层面保证每个批次的正确性，无需挑战期，实现快速提款。

---

## 1. ZK 证明系统概览

| 证明系统 | 证明大小 | 验证时间 | 可信设置 | 代表 |
|----------|---------|---------|---------|------|
| **Groth16** | ~200 bytes | ~0.5ms | 需要（电路专用） | Hermez v1 |
| **PLONK** | ~400 bytes | ~1ms | 通用（SRS） | zkSync 1.x |
| **STARK** | ~100KB | ~10ms | 无需 | StarkNet |
| **Plonky2 / Plonky3** | ~50KB | ~5ms | 无需 | Polygon zkEVM |
| **Halo2** | ~50KB | ~5ms | 无需 | Scroll |

---

## 2. zkSync Era

- **类型**：Type 4 ZK-EVM（自定义 VM）
- **证明系统**：Boojum（基于 PLONK/STARK 混合）
- **EVM 兼容**：Solidity/Vyper 源码可直接编译，但字节码层不完全等价

### 特性

- **原生账户抽象（AA）**：所有账户都是合约，支持自定义签名验证与 Paymaster
- **Hyperchains（ZK Stack）**：类似 OP Stack 的 L3/L2 框架
- **数据压缩**：状态差值（state diff）压缩，而非全量 calldata

```bash
# 部署到 zkSync Era
npx hardhat deploy --network zkSyncEra
# 或使用 foundry-zksync
forge script script/Deploy.s.sol --rpc-url $ZKSYNC_RPC --zksync
```

---

## 3. Polygon zkEVM

- **类型**：Type 3（趋向 Type 2）
- **证明系统**：Plonky2 / fflonk
- **目标**：最接近 EVM 等价的 ZK Rollup

### 特性

- **EVM 字节码兼容**：直接部署现有合约，无需修改
- **CDK（Chain Development Kit）**：类似 OP Stack，可创建自定义 ZK L2
- **AggLayer**：统一多条 CDK 链的流动性与互操作

---

## 4. StarkNet

- **类型**：Type 4（自定义 Cairo VM）
- **证明系统**：STARK（无可信设置，后量子安全）
- **语言**：Cairo（类 Rust 语法，原生 ZK 友好）

### 特性

- **STARK 安全假设强**：无需可信设置，理论上后量子安全
- **Sierra → CASM**：Cairo 编译到安全中间表示（Sierra），再到 Cairo 汇编
- **Volition**：用户可选择每笔交易的 DA 模式（L1 on-chain 或 off-chain）

```cairo
// Cairo 合约示例
#[starknet::contract]
mod Counter {
    #[storage]
    struct Storage {
        count: u128,
    }

    #[external(v0)]
    fn increment(ref self: ContractState) {
        self.count.write(self.count.read() + 1);
    }
}
```

---

## 5. Scroll

- **类型**：Type 2（接近 Type 1）
- **证明系统**：Halo2（基于 KZG 承诺）
- **目标**：完整 EVM 字节码等价，最高兼容性

### 特性

- **EVM 字节码层等价**：所有以太坊工具链（Hardhat、Foundry、Etherscan）无修改使用
- **分布式 Prover 网络**：任何人可运行 Prover 赚取收益

---

## 6. 运行 ZK Prover 的挑战

ZK 证明生成（Proving）是 ZK Rollup 最大的工程挑战：

| 挑战 | 说明 |
|------|------|
| **计算成本高** | 单批次 Proving 需要大量 CPU/GPU，时间数分钟到数十分钟 |
| **递归证明** | 将多个批次证明压缩为一个（aggregation），降低 L1 验证成本 |
| **硬件加速** | GPU（CUDA）、FPGA、ASIC 正在用于加速 Proving |
| **Proving Market** | zkSync/Scroll 等探索去中心化 Proving 市场 |

---

## 6.1 Taiko（Type 1 ZK-EVM）

- **类型**：Type 1（完整以太坊等价）
- **证明系统**：多证明者（Raiko，支持 SP1/Risc0/SGX）
- **特点**：
  - 目标是与以太坊协议完全等价，无任何修改
  - 基于 Ethereum 的 Boojum 和多证明者架构
  - 去中心化 Prover 网络，任何人可参与证明

---

## 6.2 Linea（Consensys ZK-EVM）

- **类型**：Type 2+
- **证明系统**：Gnark（Go 实现，PLONK 变体）
- **背后**：ConsenSys 开发，与 MetaMask 深度集成
- **特点**：
  - 与 MetaMask 原生集成，用户获客优势
  - 分布式 Prover 网络（Linea Prover）
  - 支持 Hardhat/Foundry 原生部署

---

## 7. 选型建议

| 需求 | 推荐 |
|------|------|
| 最高 EVM 兼容 | Scroll（Type 2）或 Polygon zkEVM |
| 完整以太坊等价 | Taiko（Type 1） |
| AA 原生支持 | zkSync Era |
| 无可信设置 + 后量子 | StarkNet |
| MetaMask 生态 | Linea |
| 基于 ZK 构建自定义 L2 | Polygon CDK 或 ZK Stack（zkSync） |

---

## 参考资源

- [zkSync 文档](https://docs.zksync.io/)
- [Polygon zkEVM 文档](https://docs.polygon.technology/zkEVM/)
- [StarkNet 文档](https://docs.starknet.io/)
- [Scroll 文档](https://docs.scroll.io/)
- [Taiko 文档](https://docs.taiko.xyz/)
- [Linea 文档](https://docs.linea.build/)
- [Vitalik：ZK-EVM 类型](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)
