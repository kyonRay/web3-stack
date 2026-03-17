# NFT（非同质化代币）

> **标签：** nft、erc-721、元数据、版税、市场

NFT 在链上表示唯一性资产，应用涵盖数字艺术、收藏品、游戏道具、身份凭证等。

---

## 1. NFT 技术基础

### EVM 标准

- **ERC-721**：每个 tokenId 唯一，独立所有权
- **ERC-1155**：单合约内多类资产，支持批量操作
- **ERC-2981**：版税标准，市场通过 `royaltyInfo()` 查询版税信息
- **ERC-721A**：批量 mint 优化（Azuki 方案）

### 元数据与存储

```json
// tokenURI 指向的 JSON 格式
{
    "name": "My NFT #1",
    "description": "一个测试 NFT",
    "image": "ipfs://QmXxx.../image.png",
    "attributes": [
        { "trait_type": "Background", "value": "Blue" },
        { "trait_type": "Rarity", "value": "Rare" }
    ]
}
```

存储选择：
- **IPFS**：内容寻址，去中心化，需 pin
- **Arweave**：永久存储，一次付费
- **链上（on-chain）**：SVG/Base64 完全链上，最去中心化但昂贵

---

## 2. Solana NFT（Metaplex）

Solana NFT = SPL Token（supply=1, decimals=0）+ Metaplex Token Metadata：

```typescript
import { createNft } from '@metaplex-foundation/mpl-token-metadata'
import { percentAmount } from '@metaplex-foundation/umi'

const { nft } = await createNft(umi, {
    name: 'My NFT',
    symbol: 'MNFT',
    uri: 'https://arweave.net/metadata.json',
    sellerFeeBasisPoints: percentAmount(5),  // 5% 版税
    isMutable: true,
    isCollection: false,
})
```

**Compressed NFT（cNFT）**：使用并发 Merkle 树（Bubblegum）大幅降低 mint 成本（万分之一），适合大规模发行。

---

## 3. Sui NFT（Kiosk）

见 [smart-contract-dev/move/sui-development.md](../smart-contract-dev/move/sui-development.md) 中的 Kiosk 章节。

---

## 4. NFT 市场合约模式

### 基本拍卖/出售流程

```solidity
// 简化的 NFT 固定价格出售
contract NFTMarketplace {
    struct Listing {
        address seller;
        uint256 price;
    }
    mapping(address => mapping(uint256 => Listing)) public listings;

    function list(address nft, uint256 tokenId, uint256 price) external {
        IERC721(nft).transferFrom(msg.sender, address(this), tokenId);
        listings[nft][tokenId] = Listing(msg.sender, price);
    }

    function buy(address nft, uint256 tokenId) external payable {
        Listing memory listing = listings[nft][tokenId];
        require(msg.value >= listing.price, "Insufficient payment");
        delete listings[nft][tokenId];

        // 版税分成
        (address royaltyReceiver, uint256 royaltyAmount) =
            IERC2981(nft).royaltyInfo(tokenId, listing.price);
        payable(royaltyReceiver).transfer(royaltyAmount);
        payable(listing.seller).transfer(listing.price - royaltyAmount);

        IERC721(nft).transferFrom(address(this), msg.sender, tokenId);
    }
}
```

---

## 5. 主流 NFT 交易市场

| 市场 | 特点 |
|------|------|
| **OpenSea** | 历史最大 NFT 市场，支持多链 |
| **Blur** | 专业交易者偏好，聚合流动性，积分激励 |
| **MagicEden** | Solana 最大，已扩展至 EVM 和比特币生态 |
| **Tensor** | Solana 专业交易平台，类 Blur |
| **Zora** | 创作者导向，免费 Mint，基于 OP Stack |

### 市场版税标准

```solidity
// ERC-2981 版税查询（所有主流市场支持）
(address receiver, uint256 royaltyAmount) = IERC2981(nftContract).royaltyInfo(tokenId, salePrice);

// 在 NFT 合约中实现
function royaltyInfo(uint256, uint256 salePrice)
    external view override returns (address, uint256)
{
    return (creator, salePrice * royaltyBps / 10000); // royaltyBps = 250 即 2.5%
}
```

---

## 参考资源

- [ERC-721 标准](https://eips.ethereum.org/EIPS/eip-721)
- [ERC-2981 版税标准](https://eips.ethereum.org/EIPS/eip-2981)
- [Metaplex 文档](https://developers.metaplex.com/)
- [OpenSea 文档](https://docs.opensea.io/)
- [Blur 文档](https://docs.blur.io/)
- [Zora Protocol](https://docs.zora.co/)
