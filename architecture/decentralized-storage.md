# Decentralized Storage

> **Date:** 2024-01-01  
> **Tags:** ipfs, arweave, storage, decentralization, nft, on-chain

## Introduction

Web3 applications must store data — metadata, images, documents, and more. Relying on centralized servers (AWS S3, CDN) defeats the purpose of a decentralized application: the server can go offline, censor content, or return different data. This article surveys the main decentralized storage options and how to choose between them.

---

## 1. The Problem with Centralized Storage in Web3

A common NFT bug: the contract stores `tokenURI = "https://api.myproject.com/nft/1"`. If the company shuts down, the JSON metadata and image disappear. The token holder owns an empty pointer.

The same risk applies to DAOs storing governance documents, DeFi protocols serving frontends, and any dApp that puts mutable URLs on-chain.

---

## 2. IPFS (InterPlanetary File System)

### 2.1 How It Works

IPFS is a **content-addressed** peer-to-peer storage network. Files are identified by their **CID (Content Identifier)** — a hash of the content itself:

```
CID = Multihash( codec( content ) )
Example: QmXoypizjW3WknFiJnKLwHCnL72vedxjQkDDP1mXWo6uco
```

Because the address *is* the hash, you cannot serve different content from the same CID — censorship resistance through content addressing.

### 2.2 Pinning and Persistence

IPFS nodes **cache** content they've accessed. To guarantee persistence, content must be **pinned** — explicitly kept on one or more nodes. If no node pins your content, it may be garbage collected.

**Pinning services:**

| Service | Free Tier | Notes |
|---------|-----------|-------|
| [Pinata](https://pinata.cloud) | 1 GB | Most popular for NFT projects |
| [web3.storage](https://web3.storage) | Free | Uses Filecoin for persistence |
| [NFT.Storage](https://nft.storage) | Free for NFTs | Built on IPFS + Filecoin |
| Self-hosted | N/A | Full control, requires infra |

### 2.3 IPFS in Practice

```javascript
import { create } from 'ipfs-http-client';

const client = create({ url: 'https://ipfs.infura.io:5001/api/v0' });

// Upload a file
const { cid } = await client.add(fileBuffer);
console.log(`ipfs://${cid}`);

// Access via gateway
// https://ipfs.io/ipfs/<cid>
// https://cloudflare-ipfs.com/ipfs/<cid>
```

### 2.4 Limitations

- **Not permanent by default** — requires active pinning.
- **Retrieval speed** depends on how many peers have the content.
- **Large files** are split into chunks with a DAG (Directed Acyclic Graph) structure.

---

## 3. Filecoin

Filecoin is an **incentivized** storage network built on top of IPFS. Storage providers are paid in FIL to prove they are storing data using cryptographic proofs:

- **Proof of Replication (PoRep):** Proves a unique copy of data is stored.
- **Proof of Spacetime (PoSt):** Proves the data is stored over time.

Deals are negotiated between clients and miners for a fixed duration. Filecoin is well-suited for large cold storage (archives, backups) but has higher latency than IPFS for retrieval.

**web3.storage** and **NFT.Storage** abstract Filecoin behind a simple API, making it the easiest path to durable IPFS storage.

---

## 4. Arweave

### 4.1 The Permaweb

Arweave uses a unique **"pay once, store forever"** economic model. Uploaders pay a one-time fee in AR tokens. The protocol calculates the fee to cover an estimated 200 years of storage using an endowment model.

Data is stored in a **blockweave** — every block includes a random historical block to prove miners are storing historical data (Proof of Access consensus).

### 4.2 Properties

| Property | Value |
|----------|-------|
| **Permanence** | Designed for 200+ years |
| **Cost model** | One-time upfront fee |
| **Content addressing** | Transaction ID (TXID) |
| **Mutability** | Immutable (transactions are final) |
| **Data retrieval** | Gateway: `https://arweave.net/<txid>` |

### 4.3 When to Use Arweave

- NFT metadata and images that must be permanently accessible.
- Protocol documentation, governance proposals.
- Archives of on-chain data (e.g., Mirror.xyz articles).

```javascript
import Arweave from 'arweave';

const arweave = Arweave.init({ host: 'arweave.net', port: 443, protocol: 'https' });

const tx = await arweave.createTransaction({ data: fileBuffer });
tx.addTag('Content-Type', 'image/png');
await arweave.transactions.sign(tx, jwk);
await arweave.transactions.post(tx);

console.log(`https://arweave.net/${tx.id}`);
```

---

## 5. On-Chain Storage

Storing data directly in Ethereum contract storage or calldata is the most secure but most expensive option.

### 5.1 Contract Storage (`SSTORE`)

- **Cost:** ~20,000 gas per 32 bytes for a new slot (~$0.50–$5 per 32 bytes depending on gas price).
- **Use for:** Critical data that contracts must read (prices, ownership, settings).
- **Not for:** Large blobs (images, documents).

### 5.2 Calldata / Event Logs

Events (`LOG`) are cheaper than storage (~375 gas base + 8 gas/byte) and are permanently part of the transaction history. However, they are **not accessible from Solidity** — they live in receipts, not state.

Use events for off-chain indexers to reconstruct history cheaply.

### 5.3 Fully On-Chain NFTs

Some projects store all artwork on-chain as SVG or base64-encoded PNGs. Examples: Loot, Nouns, On-chain Monkeys.

```solidity
function tokenURI(uint256 tokenId) public pure override returns (string memory) {
    string memory svg = _buildSVG(tokenId); // generate SVG from seed
    string memory json = Base64.encode(bytes(string(abi.encodePacked(
        '{"name":"Token #', tokenId.toString(), '","image":"data:image/svg+xml;base64,',
        Base64.encode(bytes(svg)), '"}'
    ))));
    return string(abi.encodePacked('data:application/json;base64,', json));
}
```

### 5.4 EIP-4844 Blobs

EIP-4844 (Dencun, March 2024) introduced **blob-carrying transactions** with ~128 KB per blob, priced separately. Blobs are available for ~18 days (not permanent). They are used by L2s for cheap data availability, not general storage.

---

## 6. Comparison Matrix

| Solution | Permanence | Cost | Speed | On-Chain Accessible | Best For |
|----------|-----------|------|-------|---------------------|---------|
| IPFS + Pinata | As long as pinned | Low | Medium | No | NFT metadata/images |
| IPFS + Filecoin (web3.storage) | High (paid deals) | Low | Medium | No | Durable general storage |
| Arweave | Very high (pay once) | Medium | Medium | No | Permanent archives |
| Contract storage | Permanent | Very high | Fast | Yes | Critical on-chain data |
| Event logs | Permanent (in receipts) | Low | Fast | No | Historical records |
| EIP-4844 blobs | ~18 days | Very low | Fast | No | L2 data availability |
| Centralized (S3, CDN) | Operator-dependent | Very low | Fast | No | Non-critical, dev/test |

---

## 7. Recommended Patterns

### NFT Project
```
Image + Metadata → IPFS (Pinata or NFT.Storage)
tokenURI         → ipfs://<CID>  (set immutably in contract)
Critical data    → On-chain (owner, royalties)
```

### DAO Governance
```
Proposal body    → Arweave (permanent record)
IPFS CID/TXID   → Stored in proposal on-chain for reference
Voting results  → On-chain
```

### dApp Frontend
```
Frontend static files → IPFS (served via ENS + IPFS gateway)
User data             → Ceramic / ComposeDB (mutable, IPFS-backed)
```

---

## References

- [IPFS Documentation](https://docs.ipfs.tech/)
- [Filecoin Documentation](https://docs.filecoin.io/)
- [Arweave Litepaper](https://www.arweave.org/whitepaper.pdf)
- [NFT.Storage](https://nft.storage/)
- [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)
- [Storing NFT Metadata – OpenSea Standards](https://docs.opensea.io/docs/metadata-standards)
