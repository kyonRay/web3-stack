# 隐私协议与隐私保护

> **标签：** 隐私、tornado-cash、混币器、zkp、隐私交易、匿名

区块链的透明性是双刃剑：所有交易公开可查，但也意味着用户财务行为完全暴露。隐私协议通过密码学技术实现在保持可验证性的同时隐藏交易细节。

---

## 1. 区块链隐私问题

```
公开信息（所有人可见）
├── 发送方地址
├── 接收方地址
├── 交易金额
├── 合约调用数据
└── 历史所有交易记录

私下信息（理想状态）
├── 交易双方真实身份
├── 具体金额（可选）
└── 交易目的
```

链上透明性的实际影响：
- **地址追踪**：通过分析交易图谱可关联多个地址到同一用户
- **MEV 套利**：mempool 公开，套利机器人可抢跑用户交易
- **竞争泄露**：企业链上活动对竞争对手完全可见

---

## 2. 隐私方案分类

| 方案 | 原理 | 代表 | EVM 兼容 |
|------|------|------|---------|
| **混币器** | 打断资金链路 | Tornado Cash | 是 |
| **隐私链** | 底层协议级隐私 | Monero、Zcash | 否 |
| **私密 Rollup** | ZK 证明隐藏交易 | Aztec Network | 是（EVM 上） |
| **隐形地址** | 一次性接收地址 | Stealth Address（EIP-5564） | 是 |
| **可信执行环境** | TEE 链下计算 | Secret Network | 否 |

---

## 3. Tornado Cash

Tornado Cash 是以太坊上最知名的混币协议（2019-2022 活跃）：

**工作原理：**
```
存款：用户存入固定金额 ETH（0.1/1/10/100 ETH）
      → 生成 commitment 哈希存入 Merkle Tree

提款：使用零知识证明证明"我知道某个 commitment 的 secret"
      → 合约验证证明，向新地址转账
      → 无法关联存款与提款地址
```

**技术组件：**
- **Merkle Tree**：存储所有存款 commitment
- **ZK-SNARK**：Groth16 证明系统
- **Nullifier**：防止双花（每个 note 只能提款一次）

```solidity
// Tornado Cash 核心接口（简化）
interface ITornadoCash {
    // 存款：commitment = hash(secret, nullifier)
    function deposit(bytes32 commitment) external payable;

    // 提款：零知识证明证明知道 secret
    function withdraw(
        IVerifier.Proof calldata proof,
        bytes32 root,
        bytes32 nullifierHash,
        address payable recipient,
        address payable relayer,
        uint256 fee,
        uint256 refund
    ) external payable;
}
```

**监管状态：**
- 2022 年 8 月，美国财政部 OFAC 制裁 Tornado Cash 智能合约地址
- 开发者 Roman Storm 被捕并起诉（2024 年）
- 引发关于智能合约是否可被制裁的广泛讨论
- 部分前端被下线，但合约仍在链上运行

---

## 4. 隐形地址（Stealth Address）

EIP-5564 标准化隐形地址，无需混币即可实现接收方隐私：

```
发送方：从接收方公钥生成一次性隐形地址
接收方：扫描链上公告，识别属于自己的隐形地址
第三方：无法将隐形地址关联到接收方
```

```solidity
// EIP-5564 隐形地址接口
interface IERC5564Announcer {
    event Announcement(
        uint256 indexed schemeId,
        address indexed stealthAddress,
        address indexed caller,
        bytes ephemeralPubKey,
        bytes metadata
    );

    function announce(
        uint256 schemeId,
        address stealthAddress,
        bytes calldata ephemeralPubKey,
        bytes calldata metadata
    ) external;
}
```

---

## 5. Aztec Network

Aztec 是以太坊上的 ZK Rollup，将隐私作为核心特性：

**技术特点：**
- 使用 PLONK 证明系统
- 用户余额和交易对外不可见（Shielded Pool）
- Noir 语言：专为隐私电路设计的 DSL

```noir
// Noir 隐私合约示例（简化）
fn main(
    balance: Field,
    secret: Field,
    pub recipient: Field,
    pub amount: Field,
) {
    // 证明发送方余额足够，且知道 secret
    assert(balance >= amount);
    assert(hash(balance, secret) == pub_commitment);
}
```

---

## 6. 开发中的隐私考量

### 6.1 合规隐私（满足监管）

```solidity
// 使用 Merkle Proof 实现可撤销隐私（监管机构持有主密钥）
contract CompliancePrivacy {
    // 允许监管机构在特定条件下解密交易
    mapping(bytes32 => bytes) private encryptedData;

    function submit(bytes32 commitment, bytes calldata encryptedForRegulator) external {
        encryptedData[commitment] = encryptedForRegulator;
        emit PrivateTransaction(commitment);
    }
}
```

### 6.2 MEV 保护（Flashbots Protect）

```typescript
// 通过 Flashbots Protect RPC 提交交易（不经过公开 mempool）
const provider = new ethers.JsonRpcProvider('https://rpc.flashbots.net')

// 交易不会出现在公开 mempool，防止被抢跑
const tx = await signer.sendTransaction({
  to: CONTRACT_ADDRESS,
  data: calldata,
})
```

---

## 7. 法律与合规注意事项

- **OFAC 制裁**：与制裁地址交互可能违反美国法律
- **链上追踪工具**：Chainalysis、Elliptic 等公司专门提供链上合规分析
- **旅行规则**：大额加密转账要求 VASP 收集发送/接收方信息
- **合法隐私需求**：商业机密、个人财务隐私是合理需求，技术本身中立

---

## 参考资源

- [Tornado Cash 原论文](https://tornado.cash/audits/TornadoCash_whitepaper_v1.4.pdf)
- [EIP-5564（隐形地址）](https://eips.ethereum.org/EIPS/eip-5564)
- [Aztec Network 文档](https://docs.aztec.network/)
- [Noir 语言文档](https://noir-lang.org/)
- [Flashbots 文档](https://docs.flashbots.net/)
- [Chainalysis 合规资源](https://www.chainalysis.com/)
