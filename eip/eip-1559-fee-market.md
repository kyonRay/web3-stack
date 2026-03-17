# EIP-1559: Fee Market Reform

> **Date:** 2024-01-01  
> **Tags:** eip-1559, gas, fee market, base fee, ethereum, london

## Introduction

**EIP-1559**, authored by Vitalik Buterin, Eric Conner, Rick Dudley, Matthew Slipper, and Ian Norden, was activated in the **London hard fork** (August 2021). It fundamentally redesigned Ethereum's transaction fee mechanism to improve predictability, reduce overpayment, and introduce deflationary pressure on ETH supply via fee burning.

---

## 1. The Problem with the Legacy Auction Model

Before EIP-1559, Ethereum used a **first-price auction** (also called pay-as-bid):

1. Every transaction included a `gasPrice` bid.
2. Miners selected the highest-paying transactions to maximize revenue.
3. Users had to guess the right `gasPrice` — too low meant delays; too high meant overpayment.

**Pain points:**
- High volatility in gas prices between blocks.
- Users consistently overpaid due to estimation uncertainty.
- No correlation between fee and actual network congestion.

---

## 2. EIP-1559 Mechanism

### 2.1 New Transaction Fields

EIP-1559 introduces a new transaction type (`0x02`) with two fee parameters:

| Field | Description |
|-------|-------------|
| `maxFeePerGas` | Maximum total fee per gas unit the user is willing to pay |
| `maxPriorityFeePerGas` | Maximum tip per gas unit paid to the validator ("miner tip") |

The actual fee paid per gas unit is:

```
effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)
```

### 2.2 Base Fee

The `baseFee` is a **protocol-determined** value, not set by users. It adjusts automatically each block:

```
if block gas used > target gas (50% of gas limit):
    baseFee increases by up to 12.5%
else:
    baseFee decreases by up to 12.5%
```

This creates a feedback loop: high demand → higher base fee → lower demand → lower base fee. Users can reliably predict the base fee one block ahead (it's known before the block is produced).

### 2.3 Fee Burning

The entire `baseFee` is **burned** (sent to `0x0`), permanently removing that ETH from supply. Only the priority fee (tip) goes to the validator.

```
User pays: gasUsed × effectiveGasPrice
Burned:    gasUsed × baseFee
Validator: gasUsed × priorityFee
```

---

## 3. Block Size Elasticity

EIP-1559 changed the block gas limit concept:

| Parameter | Value (approximate) |
|-----------|---------------------|
| **Target block gas** | 15,000,000 gas |
| **Max block gas** | 30,000,000 gas (2× target) |

Blocks can be up to 2× the target size to absorb sudden demand spikes, but the base fee algorithm ensures average utilization trends back toward the target over time.

---

## 4. Developer Implications

### 4.1 Sending Transactions

With ethers.js v6:

```javascript
const tx = await signer.sendTransaction({
    to: recipient,
    value: ethers.parseEther("0.1"),
    maxFeePerGas:         ethers.parseUnits("50", "gwei"), // user ceiling
    maxPriorityFeePerGas: ethers.parseUnits("2", "gwei"),  // tip
});
```

ethers.js automatically sets reasonable defaults by querying `eth_feeHistory`.

### 4.2 Gas Estimation in Contracts

Contracts cannot access `baseFee` directly in Solidity, but `block.basefee` is available:

```solidity
// Available since Solidity 0.8.7 / London fork
uint256 baseFee = block.basefee;
```

This is useful for contracts that adjust behavior based on network congestion (e.g., limiting certain operations during high-fee periods).

### 4.3 Estimating Inclusion Probability

With EIP-1559, a transaction will be included in the next block if:

```
maxFeePerGas >= nextBlockBaseFee
```

Tools like `eth_feeHistory` let you estimate the 10th/50th/90th percentile priority fees over recent blocks to calibrate `maxPriorityFeePerGas`.

---

## 5. Impact on ETH Supply (Deflationary Pressure)

Since London, more ETH has been burned than newly issued during periods of high network activity, making ETH net deflationary. Key metrics:

- **Issuance post-Merge:** ~0.3% APR (issuance to validators)
- **Burn rate:** Proportional to base fee × gas used (varies ~0.1%–2%+ APR equivalent)
- **Ultra Sound Money thesis:** When burn > issuance, ETH supply shrinks

Track live burn statistics at [ultrasound.money](https://ultrasound.money).

---

## 6. Limitations and Criticisms

| Criticism | Response |
|-----------|----------|
| Does not lower fees | Correct — EIP-1559 targets *predictability*, not lower fees. Lower fees require L2s or more block space. |
| Tips still allow front-running | True — MEV (Maximal Extractable Value) remains; address with Flashbots/PBS. |
| Legacy transactions still work | Type 0 transactions are converted: `gasPrice = maxFee = maxPriorityFee` |
| Burning benefits all ETH holders | Some miners/validators prefer full fee income |

---

## 7. Summary

| Before EIP-1559 | After EIP-1559 |
|-----------------|---------------|
| First-price auction | Base fee + tip model |
| All fees to miner | Base fee burned; tip to validator |
| Unpredictable fee | Base fee known 1 block ahead |
| Fixed block size | Elastic blocks (up to 2× target) |
| Inflationary pressure | Deflationary under high load |

---

## References

- [EIP-1559 Specification](https://eips.ethereum.org/EIPS/eip-1559)
- [Ethereum Docs – EIP-1559](https://ethereum.org/en/developers/docs/gas/#eip-1559)
- [Tim Beiko – EIP-1559 FAQ](https://tim.mirror.xyz/gmcurrently)
- [ultrasound.money – ETH burn tracker](https://ultrasound.money)
- [ethers.js v6 – Fee Data](https://docs.ethers.org/v6/api/providers/#Provider-getFeeData)
