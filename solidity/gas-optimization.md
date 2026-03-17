# Gas Optimization Techniques in Solidity

> **Date:** 2024-01-01  
> **Tags:** solidity, gas, optimization, evm, performance

## Introduction

Every operation executed by the Ethereum Virtual Machine (EVM) has a gas cost. Gas is the fee mechanism that compensates validators and protects the network from abuse. For developers, optimizing gas means cheaper transactions for users and higher competitiveness for protocols. This article covers the most impactful techniques.

---

## 1. EVM Cost Mental Model

Before optimizing, understand the relative costs:

| Operation | Gas Cost (approximate) |
|-----------|------------------------|
| `SSTORE` (new non-zero value) | 20,000 |
| `SSTORE` (update existing non-zero) | 2,900 |
| `SLOAD` (cold, EIP-2929) | 2,100 |
| `SLOAD` (warm, same tx) | 100 |
| `CALL` (with ETH, cold) | 9,000 + 2,600 |
| `LOG` + topics | 375 + 375/topic |
| `MSTORE` | 3 |
| `ADD`, `MUL`, `SUB` | 3–5 |
| Zero byte in calldata | 4 |
| Non-zero byte in calldata | 16 |

**Key insight:** Storage (`SSTORE`/`SLOAD`) is by far the most expensive category. Minimize storage reads/writes.

---

## 2. Pack Storage Variables

The EVM stores state in 32-byte slots. Variables smaller than 32 bytes can be **packed** into a single slot if declared consecutively.

```solidity
// UNOPTIMIZED – uses 3 storage slots
uint256 a;    // slot 0
uint128 b;    // slot 1 (wastes 128 bits)
uint128 c;    // slot 2 (wastes 128 bits)

// OPTIMIZED – uses 2 storage slots
uint256 a;    // slot 0
uint128 b;    // slot 1 (low 128 bits)
uint128 c;    // slot 1 (high 128 bits) – packed!
```

> ⚠️ Packing only saves gas when both variables are read/written in the same transaction. Otherwise, the masking overhead can cost more.

---

## 3. Cache Storage Variables in Memory

Every `SLOAD` from cold storage costs 2,100 gas. Reading the same variable twice costs 2,200 gas. Read once, cache in a local variable:

```solidity
// UNOPTIMIZED – 2× SLOAD
function inefficient() external view returns (uint256) {
    return totalSupply * totalSupply; // two SLOADs
}

// OPTIMIZED – 1× SLOAD + 1× MLOAD
function efficient() external view returns (uint256) {
    uint256 supply = totalSupply; // one SLOAD
    return supply * supply;        // MLOAD (cheap)
}
```

---

## 4. Use `uint256` Over Smaller Integers Where Possible

Counterintuitively, using `uint8`, `uint16`, etc. in standalone storage slots is **more expensive** than `uint256` because the EVM pads to 32 bytes anyway and must mask. Use smaller types only when packing (see §2) or when storing in arrays.

```solidity
// Slightly cheaper in isolation
uint256 counter;

// Only beneficial when packed with other small types
struct Config {
    uint64  startTime;
    uint64  endTime;
    uint128 maxAmount;
} // fits in one slot
```

---

## 5. Short-Circuit Conditions

Solidity evaluates `&&` and `||` with short-circuit semantics. Put the cheapest (or most likely to fail) condition first:

```solidity
// If isWhitelisted is cheap and often false, put it first
require(isWhitelisted[msg.sender] && block.timestamp >= startTime);
```

---

## 6. Use `calldata` Instead of `memory` for Read-Only Parameters

`calldata` reads are cheaper than copying to `memory`. For external functions that don't modify the input, always use `calldata`.

```solidity
// Cheaper for external callers
function sum(uint256[] calldata values) external pure returns (uint256 total) {
    for (uint256 i; i < values.length; ) {
        total += values[i];
        unchecked { ++i; }
    }
}
```

---

## 7. Unchecked Arithmetic for Known-Safe Operations

Solidity 0.8+ adds overflow checks (via `JUMPI`). Skip them with `unchecked` when you can mathematically guarantee safety:

```solidity
// Loop counter will never exceed array length (uint256 max is unreachable)
for (uint256 i; i < length; ) {
    // ... loop body ...
    unchecked { ++i; } // saves ~30 gas per iteration
}
```

Also note: `++i` (pre-increment) is slightly cheaper than `i++` (post-increment) because post-increment stores the old value.

---

## 8. Avoid Redundant Storage Writes

Writing `0` to a non-zero slot (`SSTORE` to zero) costs 5,000 gas but gives a 15,000 gas **refund** (capped at 20% of total tx gas in EIP-3529). Avoid patterns that write and immediately overwrite:

```solidity
// Wasteful – writes 1 then writes final value
counter = 1;
doWork();
counter = finalValue;

// Better – only write the final value
doWork();
counter = finalValue;
```

---

## 9. Use Custom Errors Instead of Require Strings

Strings in `require` are stored in bytecode (RETURNDATASIZE matters) and cost more gas than selector-based custom errors:

```solidity
// Costs gas for string storage
require(balance >= amount, "Insufficient balance");

// Cheaper at deployment and at revert
error InsufficientBalance(uint256 have, uint256 need);
if (balance < amount) revert InsufficientBalance(balance, amount);
```

---

## 10. Bitmap Storage for Boolean Flags

If you store many boolean flags per address or ID, use a single `uint256` as a **bitmap** instead of a mapping of booleans:

```solidity
// Each bool uses a full 32-byte slot: expensive
mapping(uint256 => bool) public minted;

// Bitmap: 256 booleans per slot
mapping(uint256 => uint256) private _mintedBitmap;

function isMinted(uint256 tokenId) public view returns (bool) {
    uint256 bucket = tokenId / 256;
    uint256 mask   = 1 << (tokenId % 256);
    return _mintedBitmap[bucket] & mask != 0;
}

function _setMinted(uint256 tokenId) internal {
    uint256 bucket = tokenId / 256;
    uint256 mask   = 1 << (tokenId % 256);
    _mintedBitmap[bucket] |= mask;
}
```

ERC-721A by Azuki uses this pattern to reduce NFT minting costs significantly.

---

## 11. Minimize On-Chain Data; Use Events for History

Events (`LOG`) are 8–375× cheaper per byte than storage. Store only what the contract needs to enforce logic on-chain; use events for historical data that off-chain indexers can consume.

```solidity
// Off-chain indexers can reconstruct full transfer history from events
emit Transfer(from, to, amount);
// Don't store transfer history in an on-chain array
```

---

## 12. Deployment Gas: Optimizer and `via-ir`

In `hardhat.config.ts` / `foundry.toml`:

```json
// Hardhat – solidity compiler settings
"settings": {
  "optimizer": { "enabled": true, "runs": 200 },
  "viaIR": true
}
```

- **`optimizer.runs`:** Tune for your expected call frequency. More runs → cheaper execution, costlier deployment. `200` is a common default; NFT contracts that are deployed once might use `1`.
- **`viaIR`:** Routes compilation through Yul IR for better cross-function optimizations.

---

## Summary

| Technique | Approx. Saving |
|-----------|----------------|
| Pack struct variables | Up to 20,000 gas per saved slot |
| Cache `SLOAD` in memory | ~2,000 gas per duplicate read |
| `calldata` vs `memory` | ~100–500 gas per function call |
| `unchecked` loop counter | ~30 gas per iteration |
| Custom errors | ~200 gas per revert |
| Bitmap for bool mapping | ~15,000 gas per new entry vs bool mapping |

---

## References

- [EIP-2929: Gas cost increases for state access opcodes](https://eips.ethereum.org/EIPS/eip-2929)
- [EIP-3529: Reduction in refunds](https://eips.ethereum.org/EIPS/eip-3529)
- [Solidity Docs – Optimizer](https://docs.soliditylang.org/en/latest/internals/optimizer.html)
- [ERC-721A by Azuki](https://www.erc721a.org/)
- [Foundry Gas Snapshots](https://book.getfoundry.sh/forge/gas-snapshots)
