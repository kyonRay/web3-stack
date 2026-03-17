# DeFi（去中心化金融）

> **标签：** defi、dex、借贷、闪电贷、amm、流动性

DeFi 是智能合约最成熟的应用领域，通过链上代码实现无需中介的金融服务。

---

## 1. DEX 与 AMM

**去中心化交易所（DEX）** 通过 **AMM（自动做市商）** 实现资产兑换：

```
x * y = k（恒定乘积公式，Uniswap V2）

买入 Token A：增加 Token B，减少 Token A
价格 = reserve_B / reserve_A（实时变动）
```

### 主流 DEX

| DEX | 模型 | 特点 |
|-----|------|------|
| **Uniswap V3** | 集中流动性 AMM | LP 可在价格区间内提供流动性，效率更高 |
| **Curve** | 稳定交换 AMM | 稳定币/同类资产兑换，滑点极低 |
| **Balancer** | 加权 AMM | 支持非 50/50 权重，最多 8 种资产 |
| **dYdX** | 订单簿（L2） | 适合杠杆与永续合约 |

### 合约开发要点

```solidity
// UniswapV3 精确输出兑换
ISwapRouter.ExactOutputSingleParams memory params = ISwapRouter.ExactOutputSingleParams({
    tokenIn: WETH,
    tokenOut: USDC,
    fee: 3000,           // 0.3% 手续费档位
    recipient: msg.sender,
    amountOut: 1000e6,   // 精确获得 1000 USDC
    amountInMaximum: maxETH,
    sqrtPriceLimitX96: 0,
});
uint256 amountIn = router.exactOutputSingle(params);
```

---

## 2. 借贷协议

**超额抵押借贷**：
- 用户存入抵押品（如 ETH），借出其他资产（如 USDC）
- **健康因子（Health Factor）**：抵押品价值 / 借款价值，< 1 则触发清算
- **清算**：第三方（清算人）偿还部分债务，获得折价的抵押品

```solidity
// Aave V3 借款示例
IPool(AAVE_POOL).supply(USDC, 1000e6, msg.sender, 0);  // 存 1000 USDC
IPool(AAVE_POOL).borrow(WETH, 0.5e18, 2, 0, msg.sender); // 借 0.5 ETH（变动利率）
```

---

## 3. 闪电贷

单笔交易内无抵押借款，必须在同一交易内归还：

```solidity
// Aave 闪电贷
IPool(AAVE_POOL).flashLoan(
    address(this),        // 回调接收者
    assets,              // 借出资产列表
    amounts,             // 借出数量
    interestRateModes,   // 0 = 无抵押闪电贷
    params,              // 传给回调的自定义数据
    referralCode
);

// 实现回调
function executeOperation(
    address[] calldata assets,
    uint256[] calldata amounts,
    uint256[] calldata premiums, // 手续费
    address initiator,
    bytes calldata params
) external returns (bool) {
    // 执行套利、清算等操作...

    // 归还借款 + 手续费
    for (uint i = 0; i < assets.length; i++) {
        IERC20(assets[i]).approve(AAVE_POOL, amounts[i] + premiums[i]);
    }
    return true;
}
```

---

## 4. 流动性挖矿与收益聚合器

- **流动性挖矿**：LP 向协议提供流动性，获得治理代币奖励
- **收益聚合器**（如 Yearn）：自动将资产在多个协议间轮转，追求最优收益

---

## 5. DeFi 生态主流协议

### DEX

| 协议 | 特点 |
|------|------|
| **Uniswap V3/V4** | 集中流动性 AMM，以太坊生态最大 DEX |
| **PancakeSwap** | BNB Chain 最大 DEX，也支持以太坊 |
| **Curve** | 稳定币/同类资产 AMM，极低滑点 |
| **Balancer** | 加权池，支持最多 8 种资产 |
| **dYdX** | 订单簿永续合约，v4 迁移至 Cosmos 应用链 |

### 借贷协议

| 协议 | 特点 |
|------|------|
| **Aave V3** | 最大借贷协议，多链，效率模式（E-Mode） |
| **Compound V3** | 分市场模型，单资产抵押 |
| **Morpho** | 基于 Aave/Compound 的 P2P 优化层，提升利率 |

### 衍生品 / 结构化产品

| 协议 | 特点 |
|------|------|
| **HyperLiquid** | 高性能链上永续合约，订单簿模式 |
| **Pendle** | 收益代币化，可交易未来收益（PT/YT） |
| **Synthetix** | 合成资产协议，sUSD 等合成代币 |

### LSD / 再质押

| 协议 | 特点 |
|------|------|
| **EigenLayer** | 以太坊再质押（Restaking），将 ETH 安全性扩展至 AVS |
| **Lido** | 最大流动质押，stETH |
| **Rocket Pool** | 去中心化质押，rETH |

### 稳定币协议

| 协议 | 特点 |
|------|------|
| **Sky（原 MakerDAO）** | DAI/USDS 超额抵押稳定币 |
| **Ethena** | Delta 中性合成美元 USDe，基于 stETH + ETH 空头 |
| **USDC（Circle）** | 法币抵押，多链原生（CCTP） |
| **USDT（Tether）** | 市值最大，法币抵押 |

### 预测市场

| 协议 | 特点 |
|------|------|
| **Polymarket** | 最大预测市场，USDC 结算，基于 Polygon |

---

## 6. DeFi 安全注意事项

```solidity
// 使用预言机时防止价格操纵
function getPrice() internal view returns (uint256) {
    // 不要只用现货价，使用 TWAP 或多预言机
    uint256 chainlinkPrice = getChainlinkPrice();
    uint256 twapPrice = getTWAP();

    // 价格偏差超过 5% 则 revert
    require(
        abs(chainlinkPrice - twapPrice) * 100 / chainlinkPrice < 5,
        "Price oracle discrepancy"
    );
    return chainlinkPrice;
}
```

---

## 参考资源

- [Uniswap V3 文档](https://docs.uniswap.org/contracts/v3/overview)
- [Aave 文档](https://docs.aave.com/)
- [Morpho 文档](https://docs.morpho.org/)
- [Pendle 文档](https://docs.pendle.finance/)
- [EigenLayer 文档](https://docs.eigenlayer.xyz/)
- [DeFiLlama TVL 数据](https://defillama.com/)
