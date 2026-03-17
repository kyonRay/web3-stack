# Account Model vs UTXO

> **Date:** 2024-01-01  
> **Tags:** ethereum, bitcoin, utxo, account model, state

## Introduction

Two fundamentally different models underpin how blockchains track value: Bitcoin's **Unspent Transaction Output (UTXO)** model and Ethereum's **Account** model. Understanding their differences illuminates why each chain behaves the way it does — from transaction construction to smart-contract design.

---

## 1. The UTXO Model (Bitcoin)

In the UTXO model there is no concept of a "balance". Instead, the ledger is a set of **unspent outputs** — discrete coins produced by previous transactions.

### 1.1 How a Transaction Works

A transaction **consumes** one or more UTXOs as inputs and **creates** one or more new UTXOs as outputs. The rule is simple: `sum(inputs) = sum(outputs) + fee`.

```
Input UTXO:  Alice → 1 BTC (from Tx #AA)
             Alice → 0.5 BTC (from Tx #BB)
             ─────────────────────────────
             Total in: 1.5 BTC

Output UTXO: Bob   ← 1.2 BTC  (new UTXO, unspent)
             Alice ← 0.29 BTC (change, new UTXO)
             Fee:    0.01 BTC  (to miner)
```

Each output is locked by a *scriptPubKey* (usually "pay to a public key hash"). To spend it, the owner provides a valid *scriptSig* (signature + public key).

### 1.2 Properties of UTXO

| Property | Explanation |
|----------|-------------|
| **Stateless verification** | Each transaction is self-contained; full validation requires only the referenced UTXOs |
| **Parallelizable** | Independent transactions touching different UTXOs can be validated in parallel |
| **Privacy** | Change addresses can be fresh keys, obscuring transaction graphs |
| **No native "account"** | Wallets aggregate UTXOs to compute a spendable balance |

---

## 2. The Account Model (Ethereum)

Ethereum maintains a global **state** — a mapping from address to account. Every state-changing operation updates this map.

### 2.1 Account Types

| Type | Key Fields | Purpose |
|------|-----------|---------|
| **Externally Owned Account (EOA)** | `nonce`, `balance` | Controlled by a private key |
| **Contract Account** | `nonce`, `balance`, `codeHash`, `storageRoot` | Controlled by EVM bytecode |

### 2.2 How a Transaction Works

A transaction is a signed message from an EOA that specifies:
- `to`: recipient address
- `value`: ETH to transfer
- `data`: calldata (empty for ETH transfers; ABI-encoded for contract calls)
- `nonce`: sequence number preventing replay attacks
- `gasLimit`, `gasPrice` (or `maxFeePerGas`/`maxPriorityFeePerGas` post EIP-1559)

The EVM executes the transaction, updating balances and contract storage atomically. If execution reverts, state changes are rolled back but gas is still consumed.

### 2.3 Properties of the Account Model

| Property | Explanation |
|----------|-------------|
| **Global state** | Easy to query "Alice's balance" without scanning the UTXO set |
| **Smart contracts** | Persistent storage and complex logic are natural fits |
| **Sequential nonce** | Transactions from one account must be ordered, preventing double-spends |
| **State bloat** | The global state grows unboundedly; pruning is an active research area |

---

## 3. Side-by-Side Comparison

| Dimension | UTXO (Bitcoin) | Account (Ethereum) |
|-----------|---------------|-------------------|
| **State representation** | Set of unspent outputs | Global address → account map |
| **Transaction structure** | Inputs (spent UTXOs) + outputs | From, to, value, data |
| **Double-spend prevention** | Each UTXO can only be spent once | Nonce per account |
| **Parallel validation** | High (independent UTXOs) | Lower (shared global state) |
| **Smart contract support** | Limited (Bitcoin Script) | Native (EVM) |
| **Privacy** | Better by default | Worse (all state public) |
| **Auditability** | Coin history is fully traceable | Balance history is queryable |

---

## 4. Hybrid and Extended Designs

Several projects combine elements of both models:

- **Cardano (Extended UTXO / eUTXO):** Adds on-output scripts and datum fields, enabling Plutus smart contracts while keeping UTXO parallelism.
- **Nervos CKB (Cell Model):** Generalizes UTXO; each "cell" holds arbitrary data and is governed by a lock script.
- **Fuel:** Uses a UTXO model with a Rust-based VM (FuelVM) to achieve parallelism in an Ethereum-compatible L2.

---

## 5. Developer Implications

### Ethereum (Account Model)
- A contract can read and write its own storage and query other accounts' balances directly.
- Reentrancy bugs arise because the global state is mutable mid-execution.
- Gas estimation must account for state reads (`SLOAD` is expensive).

### Bitcoin (UTXO)
- Scripts are pure functions over the spending transaction — no side effects.
- Multi-party protocols (e.g., payment channels) require careful UTXO construction.
- Covenants (proposed via opcodes like `OP_CTV`) can constrain how outputs may be spent.

---

## References

- [Bitcoin Developer Guide – Transactions](https://developer.bitcoin.org/devguide/transactions.html)
- [Ethereum: Accounts](https://ethereum.org/en/developers/docs/accounts/)
- [Understanding the UTXO Model – Nervos](https://docs.nervos.org/docs/basics/concepts/cell-model)
- [Extended UTXO – Cardano](https://iohk.io/en/research/library/papers/the-extended-utxo-model/)
