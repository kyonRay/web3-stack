# OP Stack

> **标签：** op-stack、optimism、base、bedrock、superchain

OP Stack 是 Optimism 开源的 L2 标准化框架，OP Mainnet、Base、Mode、Zora 等链均基于此构建，目标是形成**超级链（Superchain）**——共享 Bridge、Sequencer 等基础设施的 L2 网络。

---

## 1. OP Stack 架构

```
用户
  ↓
Sequencer（排序器）
  ├── 接收交易（Mempool）
  ├── 排序与执行（op-geth / op-reth）
  └── 提交批次到 L1

L1（以太坊）
  ├── BatchInbox（接收压缩交易数据）
  ├── L2OutputOracle（记录提议的 L2 状态根）
  ├── DisputeGame（欺诈证明，Cannon）
  └── OptimismPortal（存款 / 提款）
```

### 主要组件

| 组件 | 说明 |
|------|------|
| **op-node** | 共识层：跟踪 L1、推导 L2 区块、驱动执行层 |
| **op-geth** | 执行层：Geth 的 OP Stack 分支，执行 EVM |
| **op-proposer** | 定期将 L2 状态根提议到 L1 |
| **op-batcher** | 将 L2 交易批次提交到 L1（calldata 或 blob）|
| **op-challenger** | 监控并挑战错误的状态根（防欺诈） |

---

## 2. 交易生命周期

```
1. 用户发交易到 Sequencer RPC
2. Sequencer 排序、执行（op-geth）
3. op-batcher 将批次压缩后写入 L1（blob 交易）
4. op-node 从 L1 解析批次，重推导 L2 链（可信推导）
5. op-proposer 每隔一段时间将 L2 outputRoot 写入 L1
6. 用户提款：等待 7 天挑战期后，通过 OptimismPortal 提款
```

---

## 3. 欺诈证明：Cannon

Cannon 是 OP Stack 的单轮欺诈证明系统：

1. 挑战者指定争议的 L2 区块范围
2. 链上 `DisputeGame` 合约运行 MIPS 仿真（Cannon VM）
3. 在 L1 上重放单步 EVM 执行，判断状态转换是否正确

> 注：OP Stack v2 已上线 Cannon，正向多轮（interactive）欺诈证明演进。

---

## 4. 部署自定义 OP Stack L2

```bash
# 安装 Optimism Monorepo
git clone https://github.com/ethereum-optimism/optimism
cd optimism
pnpm install && pnpm build

# 生成配置
cd op-deployer
op-deployer apply \
  --l1-rpc-url https://sepolia.infura.io/v3/KEY \
  --private-key $DEPLOYER_KEY \
  --artifacts-locator op-contracts://v1.6.0
```

主要配置项：

```yaml
# deploy-config.json（关键字段）
{
  "l1ChainID": 11155111,           # Sepolia
  "l2ChainID": 42069,
  "l2BlockTime": 2,                # 出块间隔（秒）
  "finalizationPeriodSeconds": 604800,  # 7 天
  "batchInboxAddress": "0x...",
  "sequencerAddress": "0x..."
}
```

---

## 5. 超级链（Superchain）

Superchain 愿景：多条 OP Stack L2 共享：

- 相同的 L1 Bridge 合约版本
- 统一的 Superchain Registry（链标识）
- 共享 Sequencer（未来，降低去中心化 Sequencer 成本）
- 通过 `SuperchainERC20` 标准实现 L2-to-L2 原生代币转移

### Superchain 成员链

| 链 | 开发方 | 特点 |
|----|--------|------|
| **OP Mainnet** | Optimism | 官方旗舰链 |
| **Base** | Coinbase | 最大日活 L2 之一 |
| **Unichain** | Uniswap | DeFi 专用，MEV 返还机制 |
| **HSK Chain（HashKey）** | HashKey | 香港合规 Web3 平台 |
| **Zora** | Zora | NFT 创作者链 |
| **Mode** | Mode | DeFi 收益链 |
| **Cyber** | CyberConnect | 社交 L2 |

---

## 6. 关键合约地址（OP Mainnet）

| 合约 | 说明 |
|------|------|
| `OptimismPortal` | 存款入口、提款出口（L1） |
| `L1CrossDomainMessenger` | L1→L2 消息传递 |
| `L2OutputOracle` | 存储提议的 L2 状态根（L1） |
| `L1StandardBridge` | ERC-20 标准跨链桥（L1） |

---

## 参考资源

- [OP Stack 文档](https://docs.optimism.io/stack/getting-started)
- [Optimism Monorepo](https://github.com/ethereum-optimism/optimism)
- [Superchain 注册](https://github.com/ethereum-optimism/superchain-registry)
- [L2Beat — OP 生态](https://l2beat.com/?tab=summary)
