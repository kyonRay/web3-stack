# Web3 术语表

> 常用 Web3/区块链术语的中英对照与简要解释。

---

## A

| 术语 | 英文 | 解释 |
|------|------|------|
| 账户抽象 | Account Abstraction (AA) | 将签名验证逻辑从协议层移至合约层（EIP-4337）|
| ABI | Application Binary Interface | 合约函数与事件的 JSON 描述，用于编码/解码调用 |
| 关联代币账户 | Associated Token Account (ATA) | Solana 上用户持有特定代币的标准账户 |

## B

| 术语 | 英文 | 解释 |
|------|------|------|
| Blob | Blob | EIP-4844 引入的新数据类型，专为 L2 DA 设计，短期可用 |
| 区块链三难题 | Blockchain Trilemma | 去中心化、安全性、可扩展性三者难以同时兼顾 |
| 构建者 | Builder | MEV 生态中专门构建区块（打包交易）的角色 |
| 打包者 | Bundler | EIP-4337 中将多个 UserOperation 打包提交的角色 |

## C

| 术语 | 英文 | 解释 |
|------|------|------|
| Calldata | Calldata | 交易中传给合约的只读输入数据 |
| CEI | Checks-Effects-Interactions | 安全合约编写模式：先检查、再改状态、最后外部调用 |
| CPI | Cross-Program Invocation | Solana 的跨程序调用（类似 EVM 的 CALL）|
| 共识机制 | Consensus Mechanism | 节点就链的状态达成一致的规则 |

## D

| 术语 | 英文 | 解释 |
|------|------|------|
| DA | Data Availability | 数据可用性：保证区块/批次数据可被获取 |
| DELEGATECALL | DELEGATECALL | 在调用方存储上下文中执行被调用方代码（代理模式基础）|
| DeFi | Decentralized Finance | 去中心化金融 |
| 动态字段 | Dynamic Fields | Sui 中运行时向对象添加字段的机制 |

## E

| 术语 | 英文 | 解释 |
|------|------|------|
| EIP | Ethereum Improvement Proposal | 以太坊改进提案 |
| ERC | Ethereum Request for Comment | 以太坊应用级标准（EIP 的子集）|
| EVM | Ethereum Virtual Machine | 以太坊虚拟机，执行智能合约 |
| 执行层 | Execution Layer | The Merge 后，负责 EVM 执行的客户端层 |

## F

| 术语 | 英文 | 解释 |
|------|------|------|
| 终局性 | Finality | 区块被不可逆确认的状态 |
| 闪电贷 | Flash Loan | 单笔交易内的无抵押借贷，必须在同一交易内归还 |
| 欺诈证明 | Fraud Proof | Optimistic Rollup 中证明状态转换错误的机制 |

## G

| 术语 | 英文 | 解释 |
|------|------|------|
| Gas | Gas | EVM 执行操作的计量单位，防止滥用 |
| Gasper | Gasper | 以太坊 PoS 共识协议（Casper-FFG + LMD-GHOST）|
| 治理代币 | Governance Token | 赋予持有者协议投票权的代币 |

## L

| 术语 | 英文 | 解释 |
|------|------|------|
| L1 | Layer 1 | 基础区块链层（以太坊主网、Solana 等）|
| L2 | Layer 2 | 构建在 L1 上的扩容层（Arbitrum、OP Mainnet 等）|
| 流动性 | Liquidity | 资产在交易时的可用性和深度 |
| 清算 | Liquidation | 抵押不足时强制出售抵押品以偿还债务 |

## M

| 术语 | 英文 | 解释 |
|------|------|------|
| Mempool | Mempool | 待处理交易池，验证者从中选择交易打包 |
| MEV | Maximal Extractable Value | 通过控制交易顺序可提取的最大价值 |
| MPT | Merkle Patricia Trie | 以太坊状态存储的树结构 |
| 铸造 | Mint | 创建新代币或 NFT |

## O

| 术语 | 英文 | 解释 |
|------|------|------|
| 预言机 | Oracle | 将链下数据引入链上的服务 |
| 乐观 Rollup | Optimistic Rollup | 乐观假设状态有效，通过欺诈证明解决争议 |

## P

| 术语 | 英文 | 解释 |
|------|------|------|
| Paymaster | Paymaster | EIP-4337 中代付 Gas 费用的合约 |
| PDA | Program Derived Address | Solana 的程序派生地址，无私钥，由程序控制 |
| PTB | Programmable Transaction Block | Sui 的可编程交易块，一次交易可链式调用多个函数 |

## R

| 术语 | 英文 | 解释 |
|------|------|------|
| 重入 | Reentrancy | 外部调用未返回前递归进入同一合约的攻击 |
| 中继者 | Relayer | 传递跨链消息的中间角色 |
| Rollup | Rollup | 将执行移至链下、将数据/证明提交 L1 的扩容方案 |
| 版税 | Royalty | NFT 二级市场交易时向创作者支付的费用 |

## S

| 术语 | 英文 | 解释 |
|------|------|------|
| Sequencer | Sequencer | L2 中负责接收、排序和批量提交交易的角色 |
| Slashing | Slashing | 惩罚作恶验证者，没收部分质押 |
| Smart Contract | Smart Contract | 部署在链上的自动执行代码 |
| SPL Token | SPL Token | Solana 的代币程序标准（类比 ERC-20）|
| SVM | Sealevel VM | Solana 的并行智能合约执行引擎 |

## T

| 术语 | 英文 | 解释 |
|------|------|------|
| TWAP | Time-Weighted Average Price | 时间加权平均价格，防价格操纵 |
| Timelock | Timelock | 延迟执行重要操作的合约（治理 + 安全）|

## U

| 术语 | 英文 | 解释 |
|------|------|------|
| UTXO | Unspent Transaction Output | Bitcoin 的账户模型：未花费的交易输出集合 |
| UUPS | Universal Upgradeable Proxy Standard | EIP-1822 可升级代理标准，升级逻辑在实现合约中 |

## V

| 术语 | 英文 | 解释 |
|------|------|------|
| 有效性证明 | Validity Proof | ZK Rollup 中证明状态转换正确的密码学证明 |
| Validium | Validium | ZK 证明上链但数据存链下的 L2 方案 |
| VRF | Verifiable Random Function | 可验证随机函数，用于链上随机数（如 Chainlink VRF）|

## Z

| 术语 | 英文 | 解释 |
|------|------|------|
| ZKP | Zero-Knowledge Proof | 零知识证明 |
| ZK Rollup | ZK Rollup | 使用有效性证明的 Rollup 方案 |
