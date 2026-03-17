# EVM 兼容链

> **标签：** evm、bnb-chain、polygon、avalanche、hyperliquid、多链部署

除以太坊主网外，大量公链与 L2 均兼容 EVM，合约代码可直接复用，但需注意各链的差异与部署细节。

---

## 1. 主流 EVM 兼容链概览

| 链 | 类型 | 原生代币 | Chain ID | TPS | 特点 |
|----|------|---------|---------|-----|------|
| **Ethereum 主网** | L1 | ETH | 1 | ~15 | 最高安全性，去中心化基准 |
| **BNB Chain（BSC）** | L1（EVM） | BNB | 56 | ~300 | Binance 生态，费用低，中心化较高 |
| **Polygon PoS** | 侧链（EVM） | POL | 137 | ~700 | 成熟生态，费用极低 |
| **Avalanche C-Chain** | L1 子网 | AVAX | 43114 | ~4500 | 1-2s 确认，EVM 完整兼容 |
| **HyperLiquid L1** | L1（EVM） | HYPE | 999 | 高 | 高性能 DEX 链，永续合约原生 |

---

## 2. BNB Chain（BSC）

BNB Chain 是 Binance 生态的 EVM 兼容链，原名 BSC（Binance Smart Chain）：

**技术特点：**
- 共识：PoSA（权益授权证明），21 个验证者（中心化程度较高）
- 出块时间：~3 秒
- Gas token：BNB
- 与以太坊完全 EVM 兼容

**生态工具：**
- 区块浏览器：[BscScan](https://bscscan.com/)
- 跨链桥：BNB Bridge（官方）
- 主流 DEX：PancakeSwap（Uniswap V3 fork）

**部署差异：**
```bash
# Hardhat 配置 BSC
networks: {
  bsc: {
    url: "https://bsc-dataseed.binance.org/",
    chainId: 56,
    accounts: [PRIVATE_KEY],
  },
  bscTestnet: {
    url: "https://data-seed-prebsc-1-s1.binance.org:8545/",
    chainId: 97,
  }
}
```

---

## 3. Polygon PoS

Polygon PoS 是以太坊的 Plasma + PoS 侧链，拥有庞大的 DeFi 和 NFT 生态：

**技术特点：**
- 共识：Heimdall + Bor（PoS，约 100 个验证者）
- 出块时间：~2 秒
- Gas token：POL（原 MATIC）
- 检查点：每隔约 30 分钟向以太坊提交检查点

**开发注意：**
- `block.timestamp` 精度略有差异
- 高 Gas Price 的操作在 Polygon 上成本极低，需调整安全假设
- 主网 RPC：`https://polygon-rpc.com`

---

## 4. Avalanche C-Chain

Avalanche 分为三条链（X/P/C），其中 **C-Chain** 是 EVM 兼容链：

**技术特点：**
- 共识：Snowman（Avalanche 共识的线性版本）
- 最终确认：~1-2 秒（无需等待确认数）
- Gas token：AVAX
- 子网（Subnet）：自定义 EVM 链，验证者集可定制

**Avalanche 子网（Subnet）开发：**
```bash
# 使用 Avalanche CLI 创建子网
avalanche subnet create mySubnet
avalanche subnet deploy mySubnet --local
```

**Polygon CDK（类似概念）：**
Polygon CDK（Chain Development Kit）允许基于 Polygon 技术栈（ZK 或 PoS）部署自定义链：
- 支持 ZK 证明（Polygon zkEVM 技术）
- 通过 AggLayer 共享流动性
- 企业或 DeFi 协议可部署专属链

---

## 5. HyperLiquid L1

HyperLiquid 是专为高性能交易设计的 EVM 兼容 L1：

**技术特点：**
- 专为永续合约与现货交易优化
- 极高吞吐量，低延迟
- 原生代币：HYPE
- 内置订单簿（不基于 AMM）
- HyperEVM：图灵完备的 EVM 运行环境

**开发定位：**
- 主要用于构建 DeFi 衍生品协议
- 提供原生预编译合约访问链上订单簿状态

---

## 6. 多链部署最佳实践

### 6.1 统一配置（Foundry）

```toml
# foundry.toml
[rpc_endpoints]
mainnet      = "${MAINNET_RPC}"
polygon      = "${POLYGON_RPC}"
bsc          = "${BSC_RPC}"
avalanche    = "${AVALANCHE_RPC}"
arbitrum     = "${ARBITRUM_RPC}"
base         = "${BASE_RPC}"

[etherscan]
mainnet   = { key = "${ETHERSCAN_KEY}" }
polygon   = { key = "${POLYGONSCAN_KEY}", url = "https://api.polygonscan.com/api" }
bsc       = { key = "${BSCSCAN_KEY}", url = "https://api.bscscan.com/api" }
avalanche = { key = "${SNOWTRACE_KEY}", url = "https://api.snowtrace.io/api" }
```

### 6.2 部署脚本多链循环

```bash
# 逐链部署同一合约
for network in mainnet polygon bsc avalanche; do
  forge script script/Deploy.s.sol \
    --rpc-url $network \
    --broadcast \
    --verify
done
```

### 6.3 跨链注意事项

| 注意点 | 说明 |
|--------|------|
| `block.timestamp` | 各链精度不同，避免毫秒级依赖 |
| `block.number` | 各链出块速度不同，避免用 block number 做时间计算 |
| `PUSH0` 操作码 | 部分旧链不支持，编译时指定 `--evm-version paris` 以下 |
| 预编译合约 | 部分链不支持以太坊全部预编译（如 `ecRecover` 以外的） |
| Gas 上限 | 各链 block gas limit 不同，批量操作需分批 |
| 原生代币精度 | 均为 18 位，但价格差异大，注意 amount 量级 |

---

## 7. EVM 兼容链选型建议

| 场景 | 推荐链 |
|------|--------|
| 最高安全性 | Ethereum 主网 |
| 低费用 DeFi | Polygon PoS 或 Arbitrum/Base（L2） |
| 高频交易 | Avalanche C-Chain 或 HyperLiquid |
| BNB 生态 | BNB Chain |
| 自定义应用链 | Avalanche Subnet 或 Polygon CDK |

---

## 参考资源

- [BNB Chain 文档](https://docs.bnbchain.org/)
- [Polygon PoS 文档](https://docs.polygon.technology/)
- [Polygon CDK 文档](https://docs.polygon.technology/cdk/)
- [Avalanche 文档](https://docs.avax.network/)
- [HyperLiquid 文档](https://hyperliquid.gitbook.io/hyperliquid-docs/)
- [ChainList — 链 RPC 列表](https://chainlist.org/)
