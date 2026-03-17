# EIP-7594：PeerDAS（点对点数据可用性采样）

> **标签：** eip-7594、PeerDAS、数据可用性、DAS、blob 扩容、纠删码、Fusaka
> **所属升级：** Fusaka（Fulu-Osaka，主网激活：2025.12.3）
> **状态：** 已上线

---

## 1. 要解决的问题

### 1.1 EIP-4844 的带宽瓶颈

EIP-4844 引入了 blob 机制，但每个全节点必须**下载区块中的所有 blob 数据**。这个"全量下载"要求成为了进一步扩容的根本瓶颈：

```
EIP-4844（Dencun）blob 下载模型：

每个全节点
├── 接收区块头
├── 下载 blob 0（128 KB）
├── 下载 blob 1（128 KB）
├── 下载 blob 2（128 KB）
│   ...
└── 下载 blob N（128 KB）← 必须全部下载，才能验证 DA

问题：如果要将 blob 从 6 扩展到 64，
  每块需要传输：64 × 128 KB = 8 MB
  每 12 秒一块：8 MB / 12s ≈ 5.3 Mbps 持续下载
  → 家用节点完全不可能承受
```

### 1.2 数据可用性问题的本质

在 Rollup 安全模型中，"数据可用性"意味着：

```
L2 批次数据 必须 满足：
  1. 已发布到以太坊（L1 节点可以获取）
  2. 任何人都可以下载和验证

如果 L2 Sequencer 只发布一半数据：
  → 欺诈证明无法构造（缺少数据）
  → ZK 证明有效但数据不公开（黑盒）
  → 用户资产风险
  
"数据可用性攻击"：
  发布区块头（欺骗轻节点认为数据已发布）
  但实际数据被扣留
  全节点必须下载全部数据才能发现这种攻击
```

**PeerDAS 解决的核心问题**：如何在不要求每个节点下载全部数据的前提下，确保所有人都可以检测"数据是否真的被发布了"？

### 1.3 为什么需要 PeerDAS，而不是其他方案

```
方案对比：

方案 A：继续全量下载（现状）
  → 扩容受限于节点带宽，无法超过家用节点承受范围

方案 B：委托给专门的 DA 节点（Celestia、EigenDA 方案）
  → 引入额外信任假设，以太坊 L1 的安全性依赖外部系统

方案 C：数据可用性采样（DAS）
  → 每个节点只下载少量随机采样，概率性地保证 DA
  → 保留以太坊的去中心化节点参与 DA 验证
  → 这是 EIP-7594 PeerDAS 的方案
```

---

## 2. 核心变更

### 2.1 核心思想：纠删码 + 随机采样

**纠删码（Erasure Coding）**

```
原始数据：D（如 1 MB blob 数据）
纠删码编码后：2D（2 MB，原始数据的 2 倍）

魔法性质：
  如果你拥有任意 50%（1 MB）的编码数据，就可以恢复全部原始数据
  
  即：即使 50% 的数据块丢失，仍然可以重建

EIP-7594 的具体方案：
  每个 blob 展开成 4096 个字段元素
  纠删码扩展为 8192 个"cell"（每个 cell 64 字节）
  
  理论：持有任意 4096/8192 = 50% 的 cells，可以恢复完整 blob
  实践阈值：持有任意 50% 即可重建
```

**随机采样验证**

```
节点不下载所有 cells，而是：
  1. 随机选取少量 cells（如 1/8，即 512 个）
  2. 验证这些 cells 的 KZG 证明
  
概率推理：
  如果攻击者扣留了 50%+ 的数据（以至于无法恢复）：
    随机采样 512 个 cells 中，
    碰巧全部命中"公开部分"的概率：(0.5)^512 ≈ 10^-154（天文数字小的概率）

结论：任何一个节点采样 ~64 个 cells，就有 99.99999%+ 的概率检测到数据扣留攻击
```

### 2.2 列式数据分发（Column-Based Distribution）

EIP-7594 组织数据的方式是**列式**的：

```
blob 数据矩阵：

          blob_0    blob_1    blob_2    ...  blob_8
row_0   [ cell_00, cell_01, cell_02, ..., cell_08 ]
row_1   [ cell_10, cell_11, cell_12, ..., cell_18 ]
...
row_127 [ cell_1270, ...]

每列（column）包含一个区块内所有 blob 的同一位置 cell

PeerDAS 节点职责：
  每个节点订阅特定列的数据（按节点 ID 决定）
  确保自己订阅的列在网络中保持可用
  
列的总数：128 列（DATA_COLUMN_SIDECAR_SUBNET_COUNT = 128）
每个节点订阅：8 列（1/16 的数据）

全网：
  128 列 × 多个节点/列 = 高冗余，任意 50% 列即可重建
```

### 2.3 Cell KZG 证明

每个 cell 都附有一个 KZG 证明，证明该 cell 的内容与对应 blob 的 KZG 承诺一致：

```
KZG 证明格式：
  cell_proof = 48 bytes（G1 点）
  
证明内容：
  "这个 cell 的 64 字节数据确实是原始 blob 多项式在特定点集上的值"
  
验证：
  验证 cell_proof 是否对应于 blob 的 KZG 承诺（链上已发布）
  → 无法伪造（密码学保证）
  → 验证成本低（BLS12-381 预编译，EIP-2537 提供支持）
```

### 2.4 网络层变化

EIP-7594 主要是共识层（P2P 网络）的变化：

```
新的 libp2p gossip 主题：
  /eth2/BlobSidecars/blob_sidecar_{index}    # EIP-4844 的完整 blob（保留）
  /eth2/DataColumnSidecars/data_column_{i}   # PeerDAS 的列数据（新增）
  
节点行为：
  - 订阅 8 个列（基于节点 NodeID 确定性选取）
  - 转发订阅列的 DataColumnSidecar 消息
  - 随机采样其他列（用于 DAS 验证）
  
验证节点（Attesters）：
  额外要求：必须在发布证明前完成 DAS 采样
  确保：验证者实际验证了 DA，而非盲目签名
```

### 2.5 超级节点（Super Nodes）

并非所有节点都需要采样，EIP-7594 定义了**超级节点**：

```
超级节点（Super Node）：
  订阅全部 128 列（持有所有数据）
  等同于 EIP-4844 的全量下载节点
  
普通节点（Regular Node）：
  订阅 8 列（持有 1/16 数据）
  随机采样验证其他列
  
轻客户端：
  仅采样，不持有任何完整列
  依赖全节点/超级节点提供采样数据
```

---

## 3. 生效前后对比

| 维度 | PeerDAS 之前（EIP-4844） | PeerDAS 之后（EIP-7594） |
|------|------------------------|------------------------|
| **每节点下载量** | 所有 blob（全量） | 1/16 的 blob 数据（8 列） |
| **DA 验证方式** | 全量下载验证 | 随机采样（概率保证）|
| **blob 扩容上限** | 受节点带宽限制（~9 max） | 理论上可扩展到 128+ blobs |
| **Fusaka 激活时参数** | 9 blobs max（Pectra） | target 10，max 16（BPO 调整） |
| **节点加入门槛** | 高（需要全部 blob 带宽） | 低（只需 1/16 带宽） |
| **网络消息类型** | blob sidecar（完整 blob） | data column sidecar（列数据）|
| **DA 安全假设** | 至少 1 诚实全节点 | 1%+ 节点诚实采样即可检测攻击 |

### 容量提升预测

```
EIP-4844（Dencun）     ：3 target / 6 max blobs/block
EIP-7691（Pectra）     ：6 target / 9 max blobs/block
EIP-7594（Fusaka）     ：10 target（BPO 调整后）
BPO 后续分叉           ：14+ target

Full Danksharding（未来）：64-128 target blobs/block
  （需要 DAS 完全成熟 + 全量 EIP-7594 生态）

等效 L2 TPS 估算：
  当前（9 blobs）：以太坊 L2 生态合计 ~10,000 TPS
  Fusaka（10-16 blobs）：~20,000-30,000 TPS
  Full Danksharding（128 blobs）：~100,000-200,000 TPS
```

---

## 4. 影响范围

### 对 L2 开发者

- **无需代码修改**：DA 层变化对 L2 完全透明，批次提交逻辑不变
- **容量提升**：Fusaka 激活后可用 blob 空间增加，费用进一步降低
- **长期规划**：Full Danksharding 路线明确，L2 可以更激进地规划扩容

### 对节点运营者

- **普通全节点**：带宽需求变化不大（只需处理 1/16 的 blob 数据）
- **存档节点/超级节点**：需要更多带宽（处理全量 blob 数据）
- **客户端更新**：共识客户端（Prysm、Lighthouse、Teku 等）需要实现 PeerDAS 网络层

```
硬件要求变化（普通节点）：
  带宽：EIP-4844 时 ~512 KB/block → PeerDAS 后 1/16 × 扩容量 ≈ 类似
  存储：DA 数据仍然 ~18 天保留，但总量因 1/16 占比而降低
  CPU：需要验证 cell KZG 证明（依赖 EIP-2537 预编译）
```

### 对验证者

- **额外责任**：在发布证明（attestation）前需要完成 DAS 采样
- **客户端支持**：需要更新到支持 PeerDAS 的共识客户端版本

### 对以太坊协议研究者/开发者

EIP-7594 是迈向 Full Danksharding 的关键里程碑：

```
路线图：
  Phase 0（EIP-4844）：blob 机制上线，全量下载
  Phase 1（EIP-7594）：P2P 层 DAS，1/16 下载，blob 适度扩容
  Phase 2（Full Danksharding）：完整 DAS，64-128 blobs，带宽不再是限制
```

---

## 5. 与其他 EIP 的关系

| 关联 EIP | 关系 | 说明 |
|---------|------|------|
| **EIP-4844** | 直接扩展 | PeerDAS 复用 4844 的 KZG 承诺基础，在 P2P 层添加列式采样 |
| **EIP-7691** | 参数扩展 | 7691 在 Pectra 提升了 blob target/max；PeerDAS 在 Fusaka 进一步扩容 |
| **EIP-2537** | 技术依赖 | Cell KZG 证明验证依赖 BLS12-381 预编译；PeerDAS 受益于 EIP-2537 |
| **EIP-7892（BPO）** | 配套机制 | BPO 分叉机制允许在 PeerDAS 稳定后，无需全量硬分叉地调整 blob 参数 |
| **Full Danksharding** | 演进方向 | PeerDAS 是 Full Danksharding 的第一步，验证 DAS 机制的实际可行性 |
| **EIP-4337 / EIP-7702** | 无直接关联 | AA 与 DA 扩容是独立的优化维度 |

### Fusaka 后的 BPO 分叉路线

```
EIP-7892 定义的 BPO（Blob Parameter Only）分叉机制：

Fusaka 激活（2025.12）：
  PeerDAS 上线 → blob target 可以超越纯带宽限制
  BPO 首次调整：target 6→10，max 9→15
  
BPO-2（计划中，具体时间 TBD）：
  target 10→14，max 15→21
  
条件：PeerDAS 网络稳定运行，各客户端实现一致

目标：逐步提升至 Full Danksharding 的参数级别（64-128 blobs）
```

---

## 参考资源

- [EIP-7594 原文](https://eips.ethereum.org/EIPS/eip-7594)
- [Dankrad Feist：数据可用性采样 & 纠删码](https://notes.ethereum.org/@dankrad/new_sharding)
- [PeerDAS 技术规范（consensus-specs）](https://github.com/ethereum/consensus-specs/tree/dev/specs/fulu)
- [以太坊路线图：The Surge](https://ethereum.org/en/roadmap/danksharding/)
- [PeerDAS 进度跟踪](https://github.com/ethereum/pm/issues/1092)
- [Blobscan - blob 使用统计](https://blobscan.com/)
