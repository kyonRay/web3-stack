# 共识层实现

> **标签：** pos、gasper、casper-ffg、lmd-ghost、验证者、终局性

以太坊在 2022 年 9 月通过 The Merge 切换到**权益证明（PoS）**，共识协议为 **Gasper**（Casper-FFG + LMD-GHOST 的组合）。

---

## 1. 共识基本概念

| 概念 | 说明 |
|------|------|
| **Slot** | 12 秒一个时槽，每槽选出一个区块提议者 |
| **Epoch** | 32 个 Slot = 6.4 分钟，epoch 边界进行 Checkpoint 投票 |
| **Validator** | 质押 32 ETH 的参与者，负责出块与 Attest |
| **Committee** | 每个 Slot 随机分配的验证者子集，负责对区块做 Attestation |
| **Finality** | 连续两个 epoch 的 Checkpoint 获得 2/3 超多数投票后，才能最终化 |

---

## 2. Gasper = Casper-FFG + LMD-GHOST

### 2.1 LMD-GHOST（分叉选择规则）

"Latest Message Driven Greedy Heaviest Observed Sub-Tree"

- 每个验证者的最新 attestation 记录其支持的区块
- 从创世块出发，贪心选择**被验证者最新投票总权重最大**的子树
- 解决短期分叉，实现活性（liveness）

### 2.2 Casper-FFG（最终化协议）

- 验证者对每个 epoch 的 **Checkpoint**（边界区块）投票
- 当 Checkpoint C 获得 ≥ 2/3 超多数投票 → **Justified**
- 当 C 的子 Checkpoint 也 Justified 后，C → **Finalized**
- 已最终化的区块不可回滚（否则攻击者至少 1/3 质押被罚没）

---

## 3. 验证者生命周期

```
质押 32 ETH → 激活队列 → Active 状态
                                ↓
                    出块 / Attestation / Sync Committee
                                ↓
              自愿退出 or 被罚没（Slashing）→ 退出队列 → 提款
```

### 3.1 Slashing 条件

| 场景 | 说明 |
|------|------|
| **双签** | 在同一 Slot 对两个不同区块提议 |
| **环绕投票** | Attestation 环绕了之前的 Attestation |

Slashing 后：直接损失部分质押 + 后续惩罚递增（与同期被罚没的验证者数量相关）。

---

## 4. Sync Committee

- 每 256 个 epoch 随机选取 512 个验证者组成 Sync Committee
- 负责对最新区块头做聚合签名（BLS 聚合），供轻客户端验证
- Helios 等轻客户端仅需下载 Sync Committee 签名，无需全量区块

---

## 5. 共识客户端

| 客户端 | 语言 | 特点 |
|--------|------|------|
| **Prysm** | Go | 节点数量最多，文档丰富 |
| **Lighthouse** | Rust | 性能出色，安全注重 |
| **Lodestar** | TypeScript | 轻量，适合研究与集成测试 |
| **Teku** | Java | ConsenSys 开发，企业场景 |
| **Nimbus** | Nim | 超轻量，适合嵌入式/移动 |

---

## 6. 构建 L2 时与共识层的交互

L2 Sequencer 通常不直接参与以太坊共识，而是：

1. 监听 L1 的最新 finalized 区块（通过执行客户端 RPC）
2. 将 L2 批次提交到 L1 合约（作为普通 L1 交易）
3. 等待 L1 批次交易被 finalized，再声明 L2 状态最终化

---

## 参考资源

- [Ethereum PoS 官方文档](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/)
- [Gasper 论文](https://arxiv.org/abs/2003.03052)
- [Prysm 文档](https://docs.prylabs.network/)
- [Lighthouse 文档](https://lighthouse-book.sigmaprime.io/)
