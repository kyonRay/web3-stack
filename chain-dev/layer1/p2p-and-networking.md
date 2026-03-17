# P2P 网络与节点发现

> **标签：** devp2p、libp2p、discv5、区块传播、gossip

以太坊的 P2P 网络分为两层：执行层用 **devp2p**，共识层用 **libp2p**。了解网络层对链开发、节点运维与 MEV 研究都至关重要。

---

## 1. 执行层网络：devp2p

### 1.1 节点发现：discv4 / discv5

- **discv4**：Kademlia DHT 变体，UDP，用于找到其他节点的 IP:Port
- **discv5**：discv4 的继任者，支持主题（topic）广播，被共识层广泛采用

节点标识符：`enode://[publicKey]@[ip]:[port]`（执行层）/ `enr:[...]`（ENR，执行+共识层通用）

```bash
# 查看 Geth 节点 ENR
geth --exec "admin.nodeInfo.enr" attach
```

### 1.2 RLPx 传输层

- TCP 连接 + ECIES 加密握手
- 多路复用子协议（Capability）

### 1.3 以太坊子协议：ETH/68

主要消息类型：

| 消息 | 说明 |
|------|------|
| `NewBlockHashes` | 广播新区块哈希 |
| `NewBlock` | 广播完整新区块（仅向少数对等节点直接发送） |
| `Transactions` | 广播待处理交易（mempool gossip） |
| `GetBlockHeaders` | 请求一批区块头 |
| `GetBlockBodies` | 请求区块体 |
| `GetReceipts` | 请求 receipts |

区块传播策略：收到新区块后向 `√N` 个对等节点发送完整区块，向其余节点仅发送哈希。

---

## 2. 共识层网络：libp2p

共识客户端（Prysm/Lighthouse 等）基于 libp2p 构建 P2P 网络：

- **传输**：TCP + Noise 加密（或 TLS）
- **流复用**：yamux / mplex
- **PubSub**：GossipSub 协议，用于广播 Attestation、区块、Blob sidecar

### 2.1 主要 GossipSub 主题

| 主题 | 说明 |
|------|------|
| `beacon_block` | 广播新信标区块 |
| `beacon_attestation_{subnet_id}` | 按子网广播 Attestation（64 个子网） |
| `blob_sidecar_{index}` | 广播 EIP-4844 blob sidecar |
| `voluntary_exit` | 广播验证者主动退出 |
| `bls_to_execution_change` | 广播提款凭证更改 |

---

## 3. Mempool 与交易传播

- 执行客户端维护本地 **mempool**（待处理交易池）
- 新交易通过 `Transactions` 消息 gossip 给对等节点
- 出块时，Sequencer/验证者从 mempool 中选择交易（按 priorityFee 排序）

### 3.1 私有 Mempool（MEV 相关）

- **Flashbots MEV-Boost**：验证者通过中继（Relay）从构建者（Builder）处竞价获得完整区块，绕过公开 mempool
- **MEV-Share**：用户将交易回流给 searcher 换取部分 MEV 利润
- 链开发者需了解 MEV 对区块结构与排序的影响

---

## 4. 网络层优化

- **对等节点选择**：维护地理上分散的对等节点列表，降低传播延迟
- **EIP-5793（eth/68）**：交易宣告（announce）前先检查类型与大小，减少带宽浪费
- **Snap 协议**：专用于状态快照同步，比逐 trie 遍历快 10x 以上

---

## 5. 本地开发与模拟

```bash
# Anvil（Foundry）模拟网络
anvil --fork-url https://mainnet.infura.io/v3/KEY

# 多节点本地测试网（kurtosis）
kurtosis run github.com/ethpandaops/ethereum-package
```

---

## 参考资源

- [devp2p 规范](https://github.com/ethereum/devp2p)
- [libp2p 文档](https://docs.libp2p.io/)
- [以太坊网络层](https://ethereum.org/en/developers/docs/networking-layer/)
- [MEV-Boost 文档](https://docs.flashbots.net/flashbots-mev-boost/introduction)
