# GameFi（链游与游戏金融化）

> **标签：** gamefi、nft-游戏、p2e、链游、游戏资产

GameFi 将游戏（Game）与金融（Finance）结合，玩家通过游戏赚取真实价值的资产，游戏道具、角色、土地以 NFT 形式存在。

---

## 1. GameFi 核心模式

### 1.1 Play-to-Earn（P2E）

玩家通过游戏行为（战斗、建设、任务）赚取代币或 NFT：

```
玩家行为 → 链上奖励（代币/NFT） → 交易变现
```

**代表项目：**
- Axie Infinity：宠物 NFT 对战，AXS + SLP 双代币
- StepN：Move-to-Earn，跑步赚 GST
- Gods Unchained：卡牌游戏，卡牌为 NFT

### 1.2 全链游戏（Fully On-Chain Game）

游戏逻辑完全运行在链上，无中心化服务器：

```
游戏规则 → 智能合约
游戏状态 → 链上存储
随机性   → Chainlink VRF
玩家操作 → 链上交易
```

**代表项目：**
- Dark Forest：零知识证明的迷雾战争
- Loot（for Adventurers）：链上原始装备，社区二次创作

### 1.3 游戏资产 NFT 化

传统游戏道具以 NFT 形式存在，玩家真正拥有：

- **土地 NFT**：The Sandbox、Decentraland 元宇宙地块
- **角色 NFT**：游戏角色可跨平台使用（愿景）
- **道具 NFT**：武器、皮肤等稀有道具

---

## 2. GameFi 合约架构

```
游戏系统
├── 角色合约（ERC-721）— 玩家 NFT 角色
├── 道具合约（ERC-1155）— 游戏内多种道具
├── 代币合约（ERC-20）— 游戏内货币
├── 质押合约 — 锁定 NFT 参与游戏
├── 市场合约 — NFT 交易
└── 随机数（Chainlink VRF）— 开盲盒/随机掉落
```

```solidity
// 游戏资产铸造示例（ERC-1155 道具）
contract GameItems is ERC1155, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

    enum ItemType { SWORD, SHIELD, POTION }

    // 游戏服务器（链下）验证行为后铸造奖励
    function mintReward(
        address player,
        ItemType itemType,
        uint256 amount,
        bytes calldata signature
    ) external {
        // 验证游戏服务器签名（防止作弊）
        require(_verifySignature(player, itemType, amount, signature), "Invalid sig");
        _mint(player, uint256(itemType), amount, "");
    }
}
```

---

## 3. 随机性：开盲盒与掉落

GameFi 大量依赖可验证随机数（VRF）：

```solidity
import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2Plus.sol";

contract LootBox is VRFConsumerBaseV2Plus {
    mapping(uint256 => address) public requestToPlayer;

    function openBox() external returns (uint256 requestId) {
        requestId = s_vrfCoordinator.requestRandomWords(...);
        requestToPlayer[requestId] = msg.sender;
    }

    function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords) internal override {
        address player = requestToPlayer[requestId];
        uint256 rarity = randomWords[0] % 100;

        if (rarity < 60) {
            _mintCommon(player);      // 60% 普通
        } else if (rarity < 90) {
            _mintRare(player);        // 30% 稀有
        } else {
            _mintLegendary(player);   // 10% 传说
        }
    }
}
```

---

## 4. 代币经济（Tokenomics）

GameFi 经济模型是成败关键：

| 问题 | 描述 | 典型解决方案 |
|------|------|------------|
| **通胀** | 游戏代币无限产出，价格归零 | 引入消耗机制（burn） |
| **死亡螺旋** | 价格下跌 → 玩家离开 → 更多抛售 | 双代币模型、消耗设计 |
| **投机主导** | 玩家为赚钱而非娱乐进入 | 游戏性优先，降低金融属性 |

**双代币模型（以 Axie 为例）：**
```
AXS（治理代币）— 有限供应，长期价值
SLP（游戏代币）— 可无限产出，消耗通过繁殖机制平衡
```

---

## 5. 链选择建议

| 需求 | 推荐链 |
|------|--------|
| 高频操作（链上战斗） | IMX（Immutable X）、Ronin |
| NFT 为主 | 以太坊 / Polygon |
| 全链游戏 | MUD（OP Stack）、Dojo（StarkNet） |
| 低费用大规模 | Ronin、Avalanche 子网 |

---

## 6. 全链游戏框架

- **MUD**：以太坊生态全链游戏框架，自动同步链上状态到前端
- **Dojo**：StarkNet 上的 Cairo 游戏框架，支持可证明游戏
- **Keystone**：多链全链游戏开发框架

---

## 参考资源

- [Axie Infinity 白皮书](https://whitepaper.axieinfinity.com/)
- [MUD 框架文档](https://mud.dev/)
- [Dojo 引擎](https://dojoengine.org/)
- [Immutable X 文档](https://docs.immutable.com/)
- [Chainlink VRF](https://docs.chain.link/vrf)
