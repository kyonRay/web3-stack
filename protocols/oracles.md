# 预言机

> **标签：** oracle、chainlink、twap、vrf、价格喂价

预言机（Oracle）将链下数据（价格、随机数、天气等）安全地引入链上智能合约。

---

## 1. 为什么需要预言机

区块链是封闭的确定性系统，无法主动访问链外数据。若合约需要 ETH/USD 价格来清算抵押品，就需要预言机将这个价格写入链上。

**预言机问题**：如何保证链下数据在链上是可信、不可操控的？

---

## 2. Chainlink Price Feeds

Chainlink 是最主流的去中心化预言机，数据由多个独立节点聚合：

```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface public priceFeed;

    constructor() {
        // ETH/USD on Mainnet
        priceFeed = AggregatorV3Interface(0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419);
    }

    function getLatestPrice() public view returns (int256) {
        (
            uint80 roundId,
            int256 price,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        // 验证数据新鲜度（防止使用陈旧数据）
        require(updatedAt >= block.timestamp - 3600, "Price stale");
        require(price > 0, "Invalid price");

        return price; // 8 decimals（如 300000000000 = $3000）
    }
}
```

---

## 3. TWAP（时间加权平均价格）

**链上 TWAP**：通过 Uniswap V3 的累积价格计算时间窗口内的平均价：

```solidity
import "@uniswap/v3-periphery/contracts/libraries/OracleLibrary.sol";

function getTWAP(address pool, uint32 twapInterval) external view returns (uint256 price) {
    (int24 tick,) = OracleLibrary.consult(pool, twapInterval);
    price = OracleLibrary.getQuoteAtTick(
        tick, 1e18, tokenIn, tokenOut
    );
}
```

TWAP 适合防止闪电贷操纵，但时间窗口越长，价格越滞后。

---

## 4. Chainlink VRF（随机数）

见 [smart-contract-dev/evm/solidity/advanced-topics.md](../smart-contract-dev/evm/solidity/advanced-topics.md) 中的 VRF 章节。

---

## 5. Pyth Network

Pyth 是专为 DeFi 设计的高频预言机，与 Chainlink 的关键区别：

| 维度 | Chainlink | Pyth |
|------|-----------|------|
| 更新频率 | 偏差触发（0.5%+ 或心跳） | 每 400ms（近实时） |
| 数据来源 | 节点从 CEX 聚合 | 交易所、做市商直接推送 |
| 链上验证 | 直接读取 | 需 update 交易（pull 模式） |
| 适合场景 | 常规 DeFi | 高频交易、衍生品 |

```solidity
import "@pythnetwork/pyth-sdk-solidity/IPyth.sol";
import "@pythnetwork/pyth-sdk-solidity/PythStructs.sol";

contract PythConsumer {
    IPyth pyth;
    bytes32 constant ETH_USD_PRICE_ID = 0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace;

    // 调用者需传入链下获取的 priceUpdateData
    function updateAndGetPrice(bytes[] calldata priceUpdateData) external payable returns (int64 price) {
        uint fee = pyth.getUpdateFee(priceUpdateData);
        pyth.updatePriceFeeds{value: fee}(priceUpdateData);

        PythStructs.Price memory p = pyth.getPrice(ETH_USD_PRICE_ID);
        return p.price; // 带 exponent 的价格
    }
}
```

---

## 6. 预言机安全最佳实践

- **检查数据新鲜度**：`updatedAt` 不超过允许的最大延迟
- **检查价格合理性**：设置合理的价格范围验证
- **多预言机对比**：高价值合约可对比 Chainlink、TWAP、Pyth 三个来源
- **使用 TWAP 防操纵**：大额 DeFi 操作用 TWAP 而非现货价
- **避免使用 `latestAnswer()`（已弃用）**：改用 `latestRoundData()`

---

## 参考资源

- [Chainlink Data Feeds](https://docs.chain.link/data-feeds)
- [Chainlink VRF](https://docs.chain.link/vrf)
- [Pyth Network 文档](https://docs.pyth.network/)
- [Pyth Price Feed IDs](https://pyth.network/developers/price-feed-ids)
- [UMA Optimistic Oracle](https://docs.uma.xyz/)
