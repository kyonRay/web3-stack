# RWA（现实世界资产代币化）

> **标签：** rwa、现实资产、代币化、合规、国债、房地产

RWA（Real World Assets）将链下传统金融资产（国债、股票、房地产、信贷等）以代币形式引入链上，使 DeFi 协议能够获取真实收益。

---

## 1. RWA 兴起背景

2022-2023 年高利率环境下，链上美国国债年化收益（4-5%）远超大多数 DeFi 协议，推动机构资金和 DeFi 协议纷纷布局 RWA：

- **MakerDAO**：将部分储备金投入短期国债
- **Aave**：引入 RWA 作为抵押品
- **Ondo Finance**：链上美国国债基金（OUSG）
- **Centrifuge**：信贷/应收账款代币化

---

## 2. RWA 分类

| 类型 | 代表项目 | 特点 |
|------|---------|------|
| **美国国债/货币市场基金** | Ondo（OUSG/USDY）、Superstate、Mountain | 稳定收益 4-5%，合规要求高 |
| **私人信贷/应收账款** | Centrifuge、Maple Finance | 更高收益，信用风险 |
| **房地产** | RealT、Lofty.ai | 碎片化所有权，流动性低 |
| **大宗商品** | Paxos Gold（PAXG）、Tether Gold（XAUt） | 黄金代币化 |
| **股票/证券** | 受监管较严，发展受限 | 需证券牌照 |

---

## 3. 技术架构

```
链下资产（国债/信贷）
    │
    ├── 法律包装（SPV / Trust）
    │     └── 持有实际资产，向代币持有人确权
    │
    ├── 预言机 / 证明服务
    │     └── 将资产价值、收益率等数据写入链上
    │
    └── ERC-20 代币（或 ERC-1400 证券代币）
          ├── 代表对底层资产的所有权/债权
          ├── 持有者获取利息/分红
          └── 二级市场交易
```

---

## 4. 智能合约实现要点

### 4.1 ERC-1400（证券代币标准）

```solidity
// ERC-1400 支持合规转让控制
interface IERC1400 {
    // 检查是否允许转让（KYC、地域限制等）
    function canTransferByPartition(
        bytes32 _partition,
        address _from,
        address _to,
        uint256 _value,
        bytes calldata _data
    ) external view returns (bytes1 _statusCode, bytes32 _applicationCode, bytes32 _partition);

    // 强制转让（监管要求）
    function operatorTransferByPartition(
        bytes32 _partition,
        address _from,
        address _to,
        uint256 _value,
        bytes calldata _data,
        bytes calldata _operatorData
    ) external returns (bytes32);
}
```

### 4.2 收益分配合约

```solidity
// 简化的国债利息分配
contract YieldDistributor {
    IERC20 public token;      // RWA 代币
    IERC20 public usdc;       // 利息以 USDC 支付

    uint256 public yieldPerToken; // 累计每 token 应得利息

    // 管理员定期将国债利息注入
    function distributeYield(uint256 amount) external onlyAdmin {
        uint256 supply = token.totalSupply();
        yieldPerToken += amount * 1e18 / supply;
    }

    // 持有者领取利息
    function claimYield(address user) external {
        uint256 owed = (token.balanceOf(user) * yieldPerToken / 1e18) - claimed[user];
        claimed[user] += owed;
        usdc.transfer(user, owed);
    }
}
```

---

## 5. 合规要求

RWA 面临严格的合规挑战：

| 要求 | 说明 |
|------|------|
| **KYC/AML** | 大多数 RWA 代币要求持有者完成身份认证 |
| **证券法** | 部分资产（股票、债券）受证券法监管，需牌照 |
| **地域限制** | 美国 SEC 要求限制美国用户（或仅限合格投资者） |
| **转让限制** | 代币转让前需验证接收方 KYC 状态 |
| **信息披露** | 定期披露底层资产状态、审计报告 |

---

## 6. 主要项目介绍

### Ondo Finance
- **OUSG**：短期美国国债 ETF 代币，面向机构（最低 $10 万）
- **USDY**：面向散户的国债收益稳定币（部分地区不可用）
- 底层持有 BlackRock 短期国债基金

### Centrifuge
- 将应收账款、抵押贷款等信贷资产代币化
- Tinlake 协议：投资者购买高/低风险档
- 与 MakerDAO 合作，信贷 RWA 作为 DAI 抵押品

### Maple Finance
- 面向机构的链上信贷市场
- 借款方为加密做市商、交易公司
- 经历 2022 年信用危机后加强风控

---

## 7. 挑战与展望

**主要挑战：**
- 链下资产的**可信托管**（Custodian 风险）
- **预言机依赖**：实时获取链下资产价格
- **清算机制**：链下资产清算速度远慢于链上
- **监管不确定性**：各司法管辖区规则差异大

**发展趋势：**
- 传统金融机构（贝莱德 BUIDL 基金）直接发行链上代币
- 国家政府探索国债代币化（新加坡 Project Guardian 等）
- RWA 成为 DeFi 协议的重要收益来源

---

## 参考资源

- [Ondo Finance 文档](https://docs.ondo.finance/)
- [Centrifuge 文档](https://docs.centrifuge.io/)
- [RWA.xyz — RWA 数据仪表板](https://rwa.xyz/)
- [ERC-1400 规范](https://github.com/SecurityTokenStandard/EIP-Spec)
- [贝莱德 BUIDL 基金](https://www.blackrock.com/us/individual/products/buidl)
