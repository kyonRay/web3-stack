# 稳定币

> **标签：** 稳定币、usdc、dai、usdt、算法稳定币

稳定币是与法币（通常是美元）挂钩的代币，是 DeFi 的核心基础设施。

---

## 1. 稳定币类型

| 类型 | 代表 | 机制 | 风险 |
|------|------|------|------|
| **法币抵押** | USDC、USDT | 中心化机构持有等额美元 | 中心化、监管 |
| **加密超额抵押** | DAI/USDS（Sky） | 超额抵押加密资产，CDP 机制 | 清算风险 |
| **Delta 中性合成** | USDe（Ethena） | stETH + 永续空头对冲 | 资金费率风险 |
| **算法稳定（已证失败）** | UST/LUNA（已崩溃） | 算法调节供需 | 死亡螺旋 |
| **LSD 支撑** | eUSD（Lybra）| LST（如 stETH）超额抵押 | LST 风险 |

### 主流稳定币一览

| 稳定币 | 发行方 | 类型 | 市值（约） |
|--------|--------|------|----------|
| **USDT** | Tether | 法币抵押 | 最大 |
| **USDC** | Circle | 法币抵押 | 第二大，合规性强 |
| **DAI / USDS** | Sky（MakerDAO） | 超额抵押 | 最大去中心化稳定币 |
| **USDe** | Ethena | Delta 中性 | 快速增长，提供 sUSDe 质押收益 |
| **FRAX** | Frax Finance | 算法+抵押混合 | — |

---

## 2. Sky（MakerDAO）/ DAI / USDS

DAI 通过**超额抵押的 CDP（抵押债务仓位）**维持 $1 锚定：

1. 用户存入 ETH（或其他支持的抵押品）
2. 以 150%+ 抵押率借出 DAI
3. 若抵押率跌破清算阈值（如 130%），触发清算
4. 系统收取稳定费（利息），销毁 DAI，维持供需平衡

---

## 3. Circle CCTP（跨链原生 USDC）

USDC 通过 **Cross-Chain Transfer Protocol** 实现多链原生转移（非桥接包装）：
- 链 A 上销毁 USDC
- Circle 的证明服务见证
- 链 B 上铸造等额 USDC

---

## 4. 开发中的稳定币用法

```solidity
// 接受 USDC 支付（注意 6 位小数）
function pay(uint256 amount) external {
    IERC20(USDC).transferFrom(msg.sender, address(this), amount); // amount 单位 = 1e6
}

// 使用 Chainlink 获取 USDC/USD 价格（通常稳定在 1.0 但非绝对）
(, int256 price,,,) = AggregatorV3Interface(USDC_USD_FEED).latestRoundData();
```

---

## 5. Ethena / USDe

Ethena 通过 Delta 中性策略创建合成美元 USDe：

```
用户存入 stETH
  ↓
Ethena 在衍生品交易所开等额 ETH 空头（对冲价格波动）
  ↓
铸造 USDe（$1 锚定）
  ↓
质押 USDe → sUSDe（获取资金费率收益）
```

**收益来源：**
1. stETH 质押收益（~3-4%）
2. 永续合约资金费率（多头支付空头，历史正向）

**风险：**
- 资金费率长期负值时协议可能亏损
- 交易所托管风险（CEX 持有空头仓位）

---

## 参考资源

- [Sky（MakerDAO）文档](https://docs.makerdao.com/)
- [Ethena 文档](https://docs.ethena.fi/)
- [Circle CCTP 文档](https://www.circle.com/en/cross-chain-transfer-protocol)
- [Curve 3pool（稳定币兑换）](https://curve.fi/)
