# 区块链核心概念速查

> 面向已有基础知识的开发者，快速回顾关键概念。

---

## 共识机制速查

| 机制 | 代表链 | 核心思路 |
|------|--------|---------|
| **PoW（工作量证明）** | Bitcoin、旧 ETH | 算力竞争出块，矿工消耗算力 |
| **PoS（权益证明）** | 以太坊（The Merge 后）、Solana | 质押代币，随机/轮流出块 |
| **DPoS（委托权益）** | EOS | 持币者投票选出超级节点 |
| **PoA（权威证明）** | BSC、Polygon | 预授权节点出块，牺牲去中心化 |
| **BFT 变种** | Cosmos（CometBFT）、Aptos | 即时终局性，需 2/3 超多数 |
| **PoH（历史证明）** | Solana | VDF 提供可验证时间戳，加速共识 |

---

## 账户模型 vs UTXO

| | UTXO | 账户模型 |
|--|------|---------|
| **代表** | Bitcoin | Ethereum、Solana、Aptos |
| **状态** | 未花费的输出集合 | 账户余额映射 |
| **隐私** | 更高（每笔输出一个地址） | 较低（账户持续可见） |
| **并行** | 容易（不相干 UTXO 可并行） | 较难（状态冲突） |
| **合约** | 有限（Bitcoin Script） | 丰富（EVM、SVM、MoveVM） |

---

## 交易结构速查

### 以太坊交易（EIP-1559）
```
from:           发送者地址（通过签名推导）
to:             接收者地址（nil = 合约创建）
value:          ETH 数量（wei）
data:           calldata（函数调用或任意数据）
nonce:          发送者交易计数（防重放）
maxFeePerGas:   愿意支付的最高 gas 价格
maxPriorityFeePerGas: 给验证者的小费
gas:            最大 gas 限制
signature:      ECDSA 签名（v, r, s）
```

### Solana 交易
```
signatures:     签名列表（需所有必要签名者签名）
message:
  recentBlockhash: 近期区块哈希（有效期约 150 区块）
  instructions:   指令列表（每条含 programId + accounts + data）
  accountKeys:    涉及的账户公钥列表
```

---

## EVM 关键参数速查

| 参数 | 值 |
|------|-----|
| 区块目标大小 | 15M gas |
| 区块最大大小 | 30M gas |
| 出块间隔 | 12 秒 |
| 合约最大大小 | 24,576 bytes |
| 最大 callstack 深度 | 1024 |
| SSTORE 新写 gas | 20,000 |
| SLOAD 冷读 gas | 2,100 |

---

## Solana 关键参数速查

| 参数 | 值 |
|------|-----|
| 出块时间 | ~400ms |
| TPS（理论） | 50,000+ |
| 默认 CU 限制/指令 | 200,000 |
| 最大 CU/交易 | 1,400,000 |
| 最小租金豁免（0 字节） | ~0.001 SOL |
| 交易有效期 | ~150 区块（~60 秒） |

---

## 密码学基础速查

| 概念 | 说明 |
|------|------|
| **Keccak-256** | 以太坊的哈希函数（SHA-3 变体） |
| **ECDSA** | 以太坊签名算法（secp256k1 曲线） |
| **Ed25519** | Solana 签名算法（更快） |
| **BLS 聚合签名** | 以太坊 PoS 验证者使用，支持聚合 |
| **Merkle Tree** | 交易 / 状态的高效证明结构 |
| **MPT** | Merkle Patricia Trie，以太坊状态存储 |

---

## 主流公链速查

| 链 | VM | 共识 | 特点 |
|----|-----|------|------|
| **Bitcoin** | Script（有限） | PoW | 数字黄金，UTXO |
| **Ethereum** | EVM | PoS（Gasper） | 最大智能合约生态 |
| **Arbitrum** | EVM（Nitro） | Optimistic Rollup | ETH L2，TVL 最大 |
| **Solana** | SVM（Sealevel） | PoH + PoS | 高性能，低费用 |
| **Sui** | MoveVM | PoS（BFT） | 对象模型，PTB |
| **Aptos** | MoveVM | PoS（BFT） | Block-STM 并行 |
| **BNB Chain** | EVM | PoA | 高吞吐，中心化 |
| **Polygon** | EVM | PoS / ZK | 以太坊扩容 |
