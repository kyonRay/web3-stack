# EIP-2537：BLS12-381 曲线预编译

> **标签：** eip-2537、BLS12-381、预编译、密码学、ZK、签名聚合、Pectra
> **所属升级：** Pectra（Prague-Electra，主网激活：2025.5.7）
> **状态：** 已上线（历经 5 年讨论，Pectra 正式落地）

---

## 1. 要解决的问题

### 1.1 BLS12-381 曲线在以太坊中的战略地位

BLS12-381 是以太坊 PoS 共识层（信标链）使用的椭圆曲线：
- **验证者签名**：所有 ~100 万验证者使用 BLS12-381 进行签名
- **签名聚合**：BLS 签名可以高效聚合——N 个签名可以压缩为 1 个签名，验证成本不随 N 增长
- **KZG 承诺**（EIP-4844）：基于 BLS12-381 的配对运算

然而，EIP-2537 之前，在 **EVM 执行层中**使用 BLS12-381 是不可行的：

```
问题：BLS12-381 曲线运算在 Solidity 中实现的 gas 成本

BLS G1 点加法（纯 Solidity）：约 400,000-600,000 gas
BLS 配对运算（e(G1, G2)，纯 Solidity）：约 2,000,000-5,000,000 gas

实际合约限制：
  Uniswap swap ≈ 150,000 gas
  一次 BLS 验证 ≈ 数百万 gas
  = BLS 验证消耗 10-30 个 DeFi 操作的 gas
```

这意味着以下场景在 EIP-2537 之前**实际上不可行**：
- 在 EVM 中验证 Ethereum 共识层的 BLS 签名（用于跨链桥、轻客户端证明）
- 在合约中验证 BLS 聚合签名（用于高效多签）
- 验证需要配对运算的 ZK 证明（Groth16、BLS 基础的 ZK 方案）

### 1.2 为什么一直没有这个预编译

EIP-2537 的讨论历史可以追溯到 2020 年，曾多次进入升级候选名单又被推迟：

```
时间线：
  2020.8  EIP-2537 首次提交
  2021.X  Berlin 升级 → 未包含（测试向量不完整）
  2021.X  London 升级 → 未包含
  2022.X  多次讨论 → 规范仍在完善
  2024.X  Pectra 确认包含 ✓
  2025.5  Pectra 主网激活
```

主要障碍：
1. **规范复杂性**：BLS12-381 有 G1、G2 两个子群，加上配对运算，接口定义复杂
2. **测试向量**：需要大量测试用例确保各客户端实现一致
3. **优先级竞争**：每次升级都有更紧急的 EIP 占据名额

---

## 2. 核心变更

### 2.1 新增预编译合约列表

EIP-2537 在地址 `0x0b` 到 `0x13` 新增 9 个预编译合约：

| 地址 | 功能 | 输入 | 输出 | gas 成本 |
|------|------|------|------|----------|
| `0x0b` | BLS12_G1ADD | 2 个 G1 点（256 bytes） | 1 个 G1 点（128 bytes） | 500 |
| `0x0c` | BLS12_G1MUL | 1 个 G1 点 + 标量（160 bytes） | 1 个 G1 点（128 bytes） | 12,000 |
| `0x0d` | BLS12_G1MSM | k 对 (G1点, 标量) | 1 个 G1 点 | 12,000 × k（折扣） |
| `0x0e` | BLS12_G2ADD | 2 个 G2 点（512 bytes） | 1 个 G2 点（256 bytes） | 800 |
| `0x0f` | BLS12_G2MUL | 1 个 G2 点 + 标量（288 bytes） | 1 个 G2 点（256 bytes） | 45,000 |
| `0x10` | BLS12_G2MSM | k 对 (G2点, 标量) | 1 个 G2 点 | 45,000 × k（折扣） |
| `0x11` | BLS12_PAIRING | k 对 (G1点, G2点) | 布尔值（32 bytes） | 43,000×k + 65,000 |
| `0x12` | BLS12_MAP_FP_TO_G1 | 48 bytes 域元素 | G1 点 | 5,500 |
| `0x13` | BLS12_MAP_FP2_TO_G2 | 96 bytes 域元素 | G2 点 | 75,000 |

**关键术语：**
- **G1**：BLS12-381 的第一子群，点大小 48 bytes（压缩）/ 96 bytes（未压缩）
- **G2**：BLS12-381 的第二子群，点大小 96 bytes（压缩）/ 192 bytes（未压缩）
- **MSM（Multi-Scalar Multiplication）**：多标量乘法，大量点的加权求和，ZK 证明的核心操作
- **Pairing**：配对运算 e: G1 × G2 → GT，用于签名验证和 ZK 证明

### 2.2 BLS 签名验证的 Gas 变化

```
EIP-2537 之前（纯 Solidity）：
  BLS 签名验证（1 对配对）：约 2,000,000-5,000,000 gas
  实际合约限制：完全不可行（超出单笔交易 gas 限制）

EIP-2537 之后（预编译）：
  BLS 签名验证需要：
    1. 点加法：BLS12_G1ADD（500 gas）
    2. 配对验证：BLS12_PAIRING（43,000 × 2 + 65,000 = 151,000 gas）
    合计：约 200,000-300,000 gas（合理可用）

改进：成本降低约 10-20x，从"不可行"变为"可用"
```

### 2.3 实际合约使用示例

```solidity
// BLS 签名验证合约示例（简化）
contract BLSVerifier {
    // 预编译地址
    address constant BLS12_G1ADD    = address(0x0b);
    address constant BLS12_PAIRING  = address(0x11);

    /**
     * @notice 验证 BLS 签名
     * @param pubkey G1 公钥（96 bytes，未压缩）
     * @param message 消息哈希映射到 G2 点（192 bytes）
     * @param signature G2 签名点（192 bytes）
     */
    function verifyBLSSignature(
        bytes memory pubkey,     // G1 点 (96 bytes)
        bytes memory message,    // G2 点 (192 bytes) - 消息哈希
        bytes memory signature   // G2 点 (192 bytes)
    ) public view returns (bool) {
        // 配对验证：e(pubkey, message) == e(G1_generator, signature)
        // 即：e(pubkey, msg) · e(-G1, sig) == 1
        
        bytes memory input = abi.encodePacked(
            pubkey,         // G1 点（96 bytes）
            signature,      // G2 点（192 bytes）
            // 注意：实际需要 -G1_generator，此处简化
            negG1Generator, // G1 点（96 bytes）
            message         // G2 点（192 bytes）
        );
        
        (bool success, bytes memory result) = BLS12_PAIRING.staticcall(input);
        require(success, "Pairing failed");
        return abi.decode(result, (bool));
    }
}
```

```solidity
// BLS 聚合签名验证（验证 N 个签名者同意同一消息）
contract BLSAggregateVerifier {
    /**
     * @notice 验证聚合 BLS 签名（N-of-N 多签）
     * @param pubkeys N 个 G1 公钥
     * @param aggregatedSig 聚合后的 G2 签名（N 个签名压缩为 1 个）
     * @param message 共同消息的 G2 映射
     */
    function verifyAggregateSignature(
        bytes[] memory pubkeys,
        bytes memory aggregatedSig,
        bytes memory message
    ) public view returns (bool) {
        // Step 1：聚合所有公钥（G1 点相加）
        bytes memory aggregatedPubkey = pubkeys[0];
        for (uint i = 1; i < pubkeys.length; i++) {
            bytes memory input = abi.encodePacked(aggregatedPubkey, pubkeys[i]);
            (bool success, bytes memory result) = address(0x0b).staticcall(input); // G1ADD
            require(success);
            aggregatedPubkey = result;
        }
        
        // Step 2：配对验证聚合签名
        // e(aggregatedPubkey, message) == e(G1_gen, aggregatedSig)
        // ... （配对验证逻辑）
    }
    // 关键：无论有多少签名者，验证成本基本恒定！
}
```

---

## 3. 生效前后对比

| 维度 | EIP-2537 之前 | EIP-2537 之后 |
|------|--------------|--------------|
| **G1 点加法 gas** | ~400,000（Solidity 实现） | **500**（预编译）|
| **G2 点加法 gas** | ~800,000（Solidity 实现） | **800**（预编译）|
| **配对运算 gas** | ~2,000,000-5,000,000 | **151,000**（2 对配对）|
| **BLS 签名验证** | 不可行（超出 gas 限制） | 约 200,000-300,000 gas（合理可用）|
| **BLS 聚合签名验证** | 完全不可行 | 可行（成本与单次验证相近）|
| **Groth16 ZK 验证** | 部分可行（用 BN254，不用 BLS12-381） | BLS12-381 方案现在也可行 |
| **Ethereum 共识桥** | 轻客户端证明无法在 EVM 验证 | 可在合约中验证信标链签名 |
| **EVM 中 BLS 多签** | 不实际 | 100+ 签名者的聚合验证成本接近单次 |

---

## 4. 影响范围

### 对跨链桥/轻客户端开发者

```
最重要的用例：以太坊共识层证明桥

场景：以太坊 PoS 轻客户端桥接到其他链
  Ethereum L1 → 其他链（如 Solana、Cosmos、BNB Chain）
  需要在目标链的合约中验证：
    "这个以太坊信标区块是 validator committee 签名认可的"
  
  EIP-2537 之前：
    → 目标链合约无法经济地验证 BLS12-381 签名
    → 跨链桥必须依赖多签委员会（信任假设更多）
  
  EIP-2537 之后：
    → 其他 EVM 链可以在合约中直接验证以太坊 PoS 签名
    → 无信任假设的跨链桥成为可能（至少在 EVM 链之间）
```

### 对 ZK 证明系统开发者

```
影响：ZK SNARK 证明验证成本下降

Groth16（基于 BN254，已有预编译）：不变
PLONK、Halo2（基于 BLS12-381）：
  之前：需要先在链下转换到 BN254，或使用昂贵的 Solidity 实现
  之后：可以直接在 EVM 中验证 BLS12-381 基础的证明

受益：
  - 以太坊信标链的 PoS 证明（ssz_proof）可以在合约中验证
  - BLS 基础的 ZK rollup 证明直接可用
  - EigenLayer 的 AVS（主动验证服务）签名验证
```

### 对 DApp 开发者（一般场景）

- 标准 DeFi 合约：**无影响**（不使用 BLS 曲线运算）
- 多签钱包：传统 ECDSA 多签不受影响；BLS 聚合多签现在可行（但需要新的钱包标准支持）

### 对以太坊基础设施

- **Ethereum 轻客户端**：可以在 EVM 合约中验证信标链签名，推动无信任以太坊 L1 跨链桥
- **Proof of Stake 互操作**：Cosmos ICS、Polkadot 等也使用 BLS，现在可以与以太坊直接互操作

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-4844** | 技术支撑 | 4844 的 KZG 承诺基于 BLS12-381；2537 使合约可以验证 KZG 承诺 |
| **EIP-197**（BN254 配对） | 同类预编译 | 2013 年的 BN254 配对预编译，EIP-2537 是其 BLS12-381 对应版本 |
| **EIP-196**（BN254 操作） | 同类预编译 | BN254 的 G1 加法和乘法预编译 |
| **EIP-7594（PeerDAS）** | 技术协同 | PeerDAS 使用 KZG 证明；2537 预编译使合约侧可以验证这些证明 |

---

## 参考资源

- [EIP-2537 原文](https://eips.ethereum.org/EIPS/eip-2537)
- [BLS12-381 曲线介绍（Cloudflare）](https://blog.cloudflare.com/bn254-the-new-elliptic-curve-taking-the-blockchain-world-by-storm/)
- [Ethereum 信标链 BLS 签名规范](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#bls-signatures)
- [KZG 承诺与 BLS12-381](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)
- [EIP-2537 测试向量](https://github.com/ethereum/bls12-381-tests)
