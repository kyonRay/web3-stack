# Layer 1 链开发

Layer 1 是区块链的基础层，负责共识、执行、存储与 P2P 网络。本方向聚焦以太坊客户端开发、模块化执行/共识分层，以及其他主流框架（Substrate、Cosmos SDK）。

## 文档列表

| 文档 | 说明 |
|------|------|
| [ethereum-clients.md](./ethereum-clients.md) | Geth/Reth/Nethermind/Besu 执行客户端、Prysm/Lighthouse 共识客户端、质押协议 |
| [execution-layer.md](./execution-layer.md) | 执行层：EVM、状态转换、区块处理 |
| [consensus-implementation.md](./consensus-implementation.md) | 共识层：PoS Gasper、验证者、终局性 |
| [p2p-and-networking.md](./p2p-and-networking.md) | P2P 网络：devp2p、节点发现、区块传播 |
| [substrate-cosmos.md](./substrate-cosmos.md) | Substrate 与 Cosmos SDK 框架开发 |
| [eip-and-upgrades.md](./eip-and-upgrades.md) | 关键 EIP：EIP-1559、EIP-4844、EIP-4337、EIP-7702、以太坊升级路线 |

## 建议学习顺序

1. 先读 [execution-layer.md](./execution-layer.md) — 理解 EVM 与状态机
2. 再读 [consensus-implementation.md](./consensus-implementation.md) — 理解共识分层
3. 了解 [ethereum-clients.md](./ethereum-clients.md) — 客户端实现与工程实践
4. 深入 [p2p-and-networking.md](./p2p-and-networking.md) — 网络层细节
5. 了解 [eip-and-upgrades.md](./eip-and-upgrades.md) — 重要 EIP 与路线图
6. 按需学习 [substrate-cosmos.md](./substrate-cosmos.md) — 其他框架

## 关键概念

- **执行-共识分离**（The Merge 后）：执行客户端（Geth/Reth）负责 EVM 与状态；共识客户端（Prysm/Lighthouse）负责 PoS 协议。二者通过 Engine API 通信。
- **EIP-4844（Proto-Danksharding）**：为 L2 引入 Blob 数据类型，降低 DA 成本。
- **Verkle Trees**：以太坊即将采用的新状态树结构，支持无状态客户端。
