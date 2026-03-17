# 以太坊客户端

> **标签：** geth、reth、helios、执行客户端、以太坊节点

以太坊节点由**执行客户端**（Execution Client）与**共识客户端**（Consensus Client）组成，通过 **Engine API** 通信。本文聚焦执行客户端的架构、选型与开发实践。

---

## 1. The Merge 后的客户端架构

```
用户交易
    ↓
执行客户端（Geth / Reth / Nethermind / Besu / Erigon）
  ├── JSON-RPC / WebSocket
  ├── EVM 执行
  ├── 状态数据库（MPT / Verkle）
  └── Engine API ←──────────→ 共识客户端（Prysm / Lighthouse / Teku / Lodestar）
                                  ├── PoS 协议（Gasper）
                                  ├── 验证者管理
                                  └── 出块与 Attestation
```

共识客户端负责「选哪个区块」，执行客户端负责「执行区块内的交易并更新状态」。

---

## 1.1 客户端多样性

以太坊鼓励节点运营商使用不同实现，避免单一客户端主导造成全网单点故障：

| 层级 | 客户端 | 语言 | 节点占比（约） |
|------|--------|------|------------|
| **执行层** | Geth | Go | ~50% |
| | Nethermind | C# | ~25% |
| | Reth | Rust | ~15%（快速增长） |
| | Besu | Java | ~5%（企业偏好） |
| | Erigon | Go | ~5% |
| **共识层** | Prysm | Go | ~40% |
| | Lighthouse | Rust | ~35% |
| | Teku | Java | ~15%（企业偏好） |
| | Lodestar | TypeScript | ~5% |
| | Nimbus | Nim | ~5% |

---

## 2. 主流执行客户端

### 2.1 Geth（Go Ethereum）

- **语言**：Go
- **地位**：历史最久、节点数量最多，约占以太坊节点 50%+
- **架构**：
  - LevelDB/Pebble 存储状态
  - MPT（Merkle Patricia Trie）状态树
  - EVM 解释器（正在引入 EOF 支持）
- **适合**：生产节点运维、历史深度开发、工具链集成

```bash
# 启动 Geth 全节点（主网）
geth --syncmode snap \
     --http --http.api eth,net,web3 \
     --authrpc.addr localhost --authrpc.port 8551 \
     --authrpc.jwtsecret /path/to/jwt.hex
```

### 2.2 Reth（Rust Ethereum）

- **语言**：Rust
- **特点**：Paradigm 开发，高性能、模块化设计，归档节点磁盘占用显著低于 Geth
- **架构**：
  - MDBX 数据库
  - 可插拔 EVM（revm）
  - 清晰的 crate 边界，易于二次开发
- **适合**：对性能或定制有要求的场景，例如 L2 Sequencer、MEV 节点

```bash
# 启动 Reth 全节点
reth node \
  --chain mainnet \
  --authrpc.jwtsecret /path/to/jwt.hex \
  --http --http.api eth,net,web3,debug
```

### 2.3 Nethermind

- **语言**：C#（.NET）
- **特点**：企业级功能，插件架构，对 Gnosis Chain 有良好支持
- **适合**：企业节点、需要 .NET 生态集成的场景

### 2.4 Besu（Hyperledger Besu）

- **语言**：Java
- **特点**：Linux Foundation 项目，企业联盟链（Quorum 合并）功能完整
- **适合**：企业私有网络、需要权限控制的场景

### 2.5 Erigon

- **语言**：Go（从 Geth fork）
- **特点**：磁盘效率极高，归档节点占用约 2TB（vs Geth ~13TB）
- **适合**：归档节点运维、历史数据查询

### 2.6 Helios

- **语言**：Rust
- **定位**：超轻量**轻客户端**，无需下载区块，通过信标链 sync committee 验证状态
- **适合**：移动端、嵌入式、无信任 RPC 代理

---

## 2.7 主流共识客户端

| 客户端 | 语言 | 特点 |
|--------|------|------|
| **Prysm** | Go | 最早成熟的共识客户端，文档丰富 |
| **Lighthouse** | Rust | 高性能，内存占用低，Sigma Prime 开发 |
| **Teku** | Java | ConsenSys 开发，企业偏好，MEV-Boost 集成 |
| **Lodestar** | TypeScript | ChainSafe 开发，适合 JavaScript 生态研究者 |
| **Nimbus** | Nim | 资源极省，适合嵌入式/低功耗设备 |

---

## 3. Engine API

Engine API 是执行客户端与共识客户端之间的 HTTP/WebSocket 接口（JWT 认证）：

| 方法 | 说明 |
|------|------|
| `engine_newPayloadV3` | 共识客户端将新区块发给执行客户端验证 |
| `engine_forkchoiceUpdatedV3` | 共识客户端通知执行客户端最新 safe/finalized/head |
| `engine_getPayloadV3` | 执行客户端返回待出块的 payload（含 blob sidecar） |

---

## 4. 状态数据库

以太坊状态（账户余额、合约存储）存储在 **MPT（Merkle Patricia Trie）**：

- 每个账户对应一个叶子节点；合约的 `storage` 是另一棵 MPT
- 路径长度固定为 64 半字节（256 bit 哈希），导致深度较深、证明较大
- **Verkle Tree**（计划中）：将替代 MPT，支持更小的 witness，实现无状态客户端

---

## 5. 节点同步模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Snap Sync** | 下载最近状态快照 + 验证区块头 | 日常节点（推荐） |
| **Full Sync** | 从创世块执行所有交易 | 归档节点、研究 |
| **Light** | 仅区块头 + 按需请求 | 资源受限（已逐渐被淘汰） |

---

## 6. 开发与调试工具

- `cast`（Foundry）：命令行与节点交互，调用合约、发交易、读存储
- `debug_traceTransaction`（Geth/Reth）：逐步跟踪 EVM 执行
- `eth_getProof`：获取账户或存储的 Merkle 证明
- `hardhat node` / `anvil`：本地仿真节点，支持 fork 主网状态

---

## 7. 节点类型

| 节点类型 | 存储数据 | 磁盘占用 | 适用场景 |
|---------|---------|---------|---------|
| **全节点（Full）** | 最近状态 + 区块头链 | ~1TB | 日常使用，推荐默认 |
| **归档节点（Archive）** | 全历史状态 | ~13TB+ | 查询任意历史区块状态 |
| **轻节点（Light）** | 仅区块头 | 几 GB | 资源受限，安全性较低 |

---

## 8. 质押协议（Liquid Staking）

独立运行验证者需要 32 ETH，流动质押协议降低门槛：

| 协议 | 代币 | 特点 |
|------|------|------|
| **Lido** | stETH / wstETH | 最大份额（~30% 以太坊质押），中心化担忧 |
| **Rocket Pool** | rETH | 去中心化，节点运营商只需 8 ETH + RPL |
| **Frax Ether** | frxETH / sfrxETH | DeFi 集成好，双代币模型 |
| **EigenLayer** | — | 再质押（Restaking），将以太坊安全性扩展到其他协议 |

```bash
# Rocket Pool 节点运营（需 8 ETH + RPL）
rocketpool node register
rocketpool node deposit --amount 8  # 与用户提供的 8 ETH 组合成 16 ETH minipool
```

---

## 参考资源

- [Geth 文档](https://geth.ethereum.org/docs)
- [Reth 文档](https://reth.rs/)
- [Lighthouse 文档](https://lighthouse-book.sigmaprime.io/)
- [Prysm 文档](https://docs.prylabs.network/)
- [Engine API 规范](https://github.com/ethereum/execution-apis/tree/main/src/engine)
- [以太坊节点架构图](https://ethereum.org/en/developers/docs/nodes-and-clients/)
- [客户端多样性追踪](https://clientdiversity.org/)
- [Lido 文档](https://docs.lido.fi/)
- [Rocket Pool 文档](https://docs.rocketpool.net/)
