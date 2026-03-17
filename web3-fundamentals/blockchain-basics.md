# Blockchain Basics

> **Date:** 2024-01-01  
> **Tags:** blockchain, consensus, cryptography, decentralization

## Introduction

A blockchain is a distributed ledger that stores data as a chain of cryptographically linked blocks. Unlike a traditional database managed by a central authority, a blockchain is maintained collaboratively by a peer-to-peer network. This article explains the core mechanics that make blockchains work.

---

## 1. What Is a Block?

Every block in a blockchain contains three key elements:

| Field | Description |
|-------|-------------|
| **Header** | Metadata: previous block hash, timestamp, nonce, Merkle root |
| **Transactions** | An ordered list of state-changing operations |
| **Hash** | A SHA-256 (or Keccak-256 in Ethereum) fingerprint of the entire block |

Because each block's header contains the hash of the previous block, changing any historical block invalidates every subsequent block. This chain of hashes is the source of the structure's tamper-resistance.

### 1.1 The Merkle Tree

Transactions inside a block are organized in a **Merkle tree** (binary hash tree). The root of this tree — the *Merkle root* — is stored in the block header.

```
        Root Hash
       /          \
   Hash AB       Hash CD
   /    \         /    \
Hash A  Hash B  Hash C  Hash D
  |       |       |       |
 Tx A   Tx B   Tx C   Tx D
```

The Merkle tree enables **efficient proof of inclusion**: a light client can verify that a transaction is in a block by downloading only `log₂(n)` hashes instead of all `n` transactions.

---

## 2. Consensus Mechanisms

Consensus is the process by which network participants agree on the canonical state of the ledger. Two dominant families exist:

### 2.1 Proof of Work (PoW)

Miners compete to find a nonce such that `Hash(block header) < target`. This is computationally expensive by design.

- **Security model:** An attacker needs >50% of the network's hash power to rewrite history.
- **Energy cost:** High — major criticism and motivation for alternatives.
- **Examples:** Bitcoin, Litecoin, pre-Merge Ethereum.

### 2.2 Proof of Stake (PoS)

Validators lock up (stake) cryptocurrency as collateral and are pseudo-randomly selected to propose and attest to blocks.

- **Security model:** An attacker needs to control >33% of staked value; slashing penalizes dishonest validators.
- **Energy cost:** ~99.95% lower than PoW.
- **Examples:** Ethereum (post-Merge), Cosmos, Solana.

### 2.3 Other Mechanisms

| Mechanism | Key Idea | Examples |
|-----------|----------|----------|
| Delegated PoS (DPoS) | Token holders vote for a small set of delegates | EOS, Tron |
| Proof of Authority (PoA) | Validators are pre-approved identities | Polygon PoA, private chains |
| Proof of History (PoH) | Cryptographic clock orders events before consensus | Solana |
| Byzantine Fault Tolerance (BFT) | Leader-based with instant finality | Tendermint/Cosmos |

---

## 3. The Decentralized Network Model

A public blockchain network is a **peer-to-peer (P2P) overlay network** where every node maintains its own copy of the ledger.

### 3.1 Node Types (Ethereum example)

| Node Type | Function |
|-----------|----------|
| **Full node** | Downloads and validates every block and state transition |
| **Light node** | Stores only block headers; relies on full nodes for data |
| **Archive node** | Stores full history including all intermediate states |
| **Validator node** | Participates in consensus (PoS) |

### 3.2 Network Propagation

New transactions and blocks are broadcast using a **gossip protocol**: each node relays data to a subset of its peers, which in turn relay to their peers, until the information spreads across the network (typically in a few seconds for Ethereum).

---

## 4. Cryptographic Foundations

### 4.1 Hash Functions

Blockchain heavily relies on **collision-resistant** hash functions:
- **Bitcoin:** SHA-256
- **Ethereum:** Keccak-256 (a variant of SHA-3)

Properties: deterministic, fast to compute, infeasible to reverse, small input change → completely different output (avalanche effect).

### 4.2 Digital Signatures (ECDSA)

Account ownership is proven via **Elliptic Curve Digital Signature Algorithm (ECDSA)** on the `secp256k1` curve:
1. A private key `k` is a random 256-bit number.
2. A public key `K = k × G` (elliptic curve point multiplication).
3. An Ethereum address is the last 20 bytes of `Keccak256(K)`.
4. To sign a transaction, the sender produces `(r, s, v)` — only the private key holder can do this, but anyone can verify using the public key.

---

## 5. Finality

**Finality** is the guarantee that a committed transaction cannot be reversed.

| Type | Description | Example |
|------|-------------|---------|
| **Probabilistic** | Older blocks are harder to reverse (more work on top) | Bitcoin, PoW |
| **Economic finality** | Reverting would require sacrificing staked assets | Ethereum PoS |
| **Instant finality** | Blocks are final once ≥ 2/3 of validators sign | Tendermint BFT |

In Ethereum's Gasper consensus (Casper FFG + LMD-GHOST), a checkpoint (epoch boundary block) reaches **finality** once it is justified by two consecutive supermajority votes of validators.

---

## 6. Summary

| Concept | Key Takeaway |
|---------|-------------|
| Block structure | Header + transactions, chained by hash |
| Merkle tree | Enables efficient transaction inclusion proofs |
| PoW vs PoS | Security trade-offs vs energy efficiency |
| P2P network | No central authority; gossip propagation |
| ECDSA | Private key → address; ownership without identity |
| Finality | How irreversible a transaction becomes over time |

---

## References

- [Bitcoin Whitepaper – Satoshi Nakamoto](https://bitcoin.org/bitcoin.pdf)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Ethereum Proof-of-Stake FAQ](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/)
- [Merkle Trees – Wikipedia](https://en.wikipedia.org/wiki/Merkle_tree)
