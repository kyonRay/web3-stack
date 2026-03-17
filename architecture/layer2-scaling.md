# Layer 2 Scaling Solutions

> **Date:** 2024-01-01  
> **Tags:** layer2, rollup, optimistic, zk, state-channels, scaling, ethereum

## Introduction

Ethereum Layer 1 (L1) is intentionally limited in throughput (~15–30 TPS) to preserve decentralization and security. **Layer 2 (L2) scaling** moves computation off-chain while inheriting L1 security for settlement and dispute resolution. This article surveys the major L2 architectures: Rollups (Optimistic and ZK), State Channels, Plasma, and Validiums.

---

## 1. The Blockchain Trilemma

The trilemma (attributed to Vitalik Buterin) states that a blockchain can optimize for at most two of three properties simultaneously:

```
         Decentralization
              /\
             /  \
            /    \
           /      \
          /________\
     Security    Scalability
```

L1 prioritizes security + decentralization. L2s extend scalability by adding a separate execution environment with a trust-minimized bridge back to L1.

---

## 2. Optimistic Rollups

### 2.1 How They Work

1. A **sequencer** collects transactions off-chain and batches them.
2. The sequencer posts a compressed transaction batch + new state root to L1.
3. The state root is assumed valid (**optimistically**) unless challenged.
4. During the **challenge window** (7 days on Optimism/Arbitrum), any party can submit a **fraud proof** to prove the state transition was invalid.
5. If fraud is proven, the invalid batch is reverted and the sequencer is slashed.

### 2.2 Fraud Proofs

| Type | Description | Used By |
|------|-------------|---------|
| **Multi-round interactive** | Binary search narrows dispute to single instruction | Arbitrum (Classic/Nitro) |
| **Single-round** | Re-execute the entire disputed block on L1 | Optimism (Cannon, in progress) |

### 2.3 Key Properties

| Property | Value |
|----------|-------|
| EVM compatibility | Full (bytecode compatible) |
| Withdrawal delay | 7 days (challenge window) |
| Data posted to L1 | Compressed transaction data |
| Trust assumption | At least 1 honest validator online |
| TPS (approx.) | 500–4,000 |

### 2.4 Major Implementations

- **Arbitrum One** – Multi-round fraud proofs; Nitro architecture; largest L2 by TVL.
- **OP Mainnet (Optimism)** – Single-round proofs (Cannon); OP Stack shared with Base, Mode, and others.
- **Base** – Coinbase-operated OP Stack chain; deep Coinbase wallet integration.

---

## 3. ZK Rollups

### 3.1 How They Work

1. A **prover** (off-chain) executes transactions and generates a **validity proof** (SNARK or STARK).
2. The proof cryptographically guarantees that the new state root is the correct result of executing the batch.
3. The proof is verified on L1 in a single smart contract call — no challenge window needed.

### 3.2 Proof Systems

| Proof System | Proof Size | Verification Time | Trusted Setup | Used By |
|-------------|-----------|-------------------|---------------|---------|
| **Groth16 (SNARK)** | ~200 bytes | ~0.5 ms | Required | zkSync Era (early), Hermez |
| **PLONK (SNARK)** | ~400 bytes | ~1 ms | Universal | Various |
| **STARK** | ~100 KB | ~10 ms | Not required | StarkNet, StarkEx |
| **Plonky2 / Plonky3** | ~50 KB | ~5 ms | Not required | Polygon zkEVM (partially) |

### 3.3 ZK-EVM Compatibility Spectrum

Running the EVM inside a ZK circuit is hard. Projects exist on a compatibility spectrum:

| Type | Description | Trade-off |
|------|-------------|-----------|
| **Type 1** | Fully Ethereum-equivalent (same bytecode, same hashes) | Slowest proving |
| **Type 2** | EVM-equivalent (different internal representation) | Slow proving |
| **Type 2.5** | EVM-compatible but different gas costs | Moderate |
| **Type 3** | Mostly EVM-compatible (minor differences) | Faster proving |
| **Type 4** | Compiles Solidity/Vyper to ZK-friendly VM | Fastest, least compatible |

*Classification by Vitalik Buterin, 2022.*

### 3.4 Major Implementations

- **zkSync Era (ZKsync)** – Type 4 (custom VM, Solidity/Vyper support via compiler).
- **Polygon zkEVM** – Type 3/2, pursuing full Type 2 compatibility.
- **StarkNet** – Type 4 (Cairo VM); requires Solidity→Cairo transpilation.
- **Scroll** – Type 2/1, fully bytecode compatible.
- **Linea** – Type 3, developed by ConsenSys.

---

## 4. State Channels

### 4.1 How They Work

Two or more parties lock funds in an L1 smart contract and exchange signed state updates off-chain. Only the final state is settled on L1.

```
Alice & Bob lock 10 ETH on L1
   │
   ▼
Off-chain: sign 50 rounds of moves, each supersedes the last
   │
   ▼
Either party submits latest signed state → L1 releases funds
```

### 4.2 Properties

| Property | Value |
|----------|-------|
| **Finality** | Instant (off-chain) |
| **Gas cost** | 2 on-chain txs (open + close) |
| **EVM compatibility** | N/A (state is application-specific) |
| **Liveness requirement** | Participants must be online to dispute |

### 4.3 Use Cases & Limitations

✅ **Ideal for:** Micropayments, gaming moves, high-frequency bilateral interactions.  
❌ **Not suitable for:** Open participation (anyone can join), complex multi-party state, infrequent users.

**Notable implementations:** Raiden (ERC-20 payments), Lightning Network (Bitcoin).

---

## 5. Plasma

Plasma (proposed by Vitalik Buterin and Joseph Poon, 2017) uses a hierarchical chain where periodic state commitments are posted to L1. Users exit by providing Merkle proofs.

**Status:** Largely superseded by rollups due to:
- Complex exit games for ERC-20/NFTs.
- Data availability problems (withheld exit data).
- Not EVM-compatible.

Plasma chains remain in use for specific applications (e.g., OMG Network for ETH/ERC-20 transfers).

---

## 6. Validium

A validium is similar to a ZK rollup but stores transaction data **off-chain** (not on L1):

| | ZK Rollup | Validium |
|--|-----------|---------|
| **Proof** | Validity proof on L1 | Validity proof on L1 |
| **Data availability** | On L1 (calldata/blob) | Off-chain (DAC or IPFS) |
| **Security** | Inherits full L1 DA | Trust in data availability committee |
| **Cost** | Higher (DA costs) | Lower (no on-chain DA) |

**Volition** is a hybrid that lets users choose per-transaction: rollup-mode (on-chain DA) or validium-mode (off-chain DA). Implemented by StarkEx (dYdX v3, Immutable X).

---

## 7. EIP-4844: Proto-Danksharding (Blobs)

**EIP-4844** (activated in the Dencun upgrade, March 2024) introduced **blob transactions** — a new transaction type carrying up to 128 KB of data that is:
- Available to L2 contracts for ~18 days (not permanently stored).
- Priced with a separate blob fee market (much cheaper than calldata).

This reduced L2 transaction costs by 10–100× on Optimistic and ZK rollups that post data to L1.

---

## 8. Comparison Table

| | Optimistic Rollup | ZK Rollup | State Channel | Plasma |
|--|------------------|----------|---------------|--------|
| **EVM support** | Full | Partial–Full | N/A | No |
| **Withdrawal time** | 7 days | Minutes | Instant | Days–weeks |
| **Trust model** | 1-of-N honest validator | Cryptographic proof | Liveness of participants | Data availability |
| **Best for** | General dApps | High-frequency, privacy | Micropayments | Token transfers |
| **Data on L1** | Yes (compressed) | Yes (compressed) | No | Root hashes only |

---

## References

- [An Incomplete Guide to Rollups – Vitalik Buterin](https://vitalik.eth.limo/general/2021/01/05/rollup.html)
- [EIP-4844: Shard Blob Transactions](https://eips.ethereum.org/EIPS/eip-4844)
- [The Different Types of ZK-EVMs – Vitalik Buterin](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)
- [Arbitrum Documentation](https://docs.arbitrum.io/)
- [Optimism Documentation](https://docs.optimism.io/)
- [L2Beat – Live L2 Metrics](https://l2beat.com/)
