# Smart Contract Audit AI Prompts

> **Date:** 2024-01-01  
> **Tags:** ai, prompts, audit, security, smart contracts, llm

## Introduction

Large Language Models (LLMs) can assist in the smart contract audit process by identifying common vulnerabilities, suggesting improvements, and explaining complex code logic. This article provides a collection of curated prompts for AI-assisted smart contract security review.

> ⚠️ **Important:** AI-assisted auditing is a supplement to — not a replacement for — professional human security audits. Always have high-value contracts reviewed by certified auditors.

---

## 1. Initial Security Scan Prompt

Use this prompt to get a broad security overview of a contract:

```
You are an expert smart contract security auditor with deep knowledge of Solidity, EVM internals, and common vulnerability patterns (SWC Registry, Consensys best practices, OpenZeppelin guidelines).

Analyze the following Solidity contract for security vulnerabilities. For each issue found:
1. Name the vulnerability class (e.g., reentrancy, integer overflow, access control).
2. Identify the exact function and line(s) affected.
3. Explain the attack vector step-by-step.
4. Rate the severity: Critical / High / Medium / Low / Informational.
5. Provide a code fix.

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 2. Reentrancy-Focused Audit Prompt

```
You are a Solidity security expert specializing in reentrancy attacks.

Review the following smart contract specifically for:
- Classic reentrancy (external calls before state updates)
- Cross-function reentrancy (state shared across functions)
- Cross-contract reentrancy (shared state with another contract)
- Read-only reentrancy (view functions that can be called mid-execution to read inconsistent state)

For each vulnerability:
- Quote the exact vulnerable code
- Describe the full attack scenario including malicious contract code
- Provide a CEI-compliant fix or recommend using OpenZeppelin's ReentrancyGuard

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 3. Access Control Review Prompt

```
Review the following Solidity contract for access control weaknesses.

Check for:
1. Functions that should be restricted but lack modifiers
2. Use of tx.origin instead of msg.sender for authentication
3. Missing two-step ownership transfer
4. Role escalation vulnerabilities (can a lower-privilege role grant higher privileges?)
5. Missing events for privileged actions
6. Admin key centralization risks

For each issue, specify the function name, the missing control, and the recommended fix.

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 4. Gas Optimization Review Prompt

```
You are a Solidity gas optimization expert. Analyze the following contract and identify all gas optimization opportunities.

For each optimization:
1. Quote the current code.
2. Explain why it wastes gas (reference EVM opcodes if relevant).
3. Provide the optimized version.
4. Estimate the gas saving (per call or per deployment).

Focus areas:
- Storage packing (struct layout)
- Redundant SLOADs (cache in memory)
- Loop optimizations (unchecked counter, calldata vs memory)
- Custom errors vs require strings
- Unnecessary storage writes
- Event vs storage for historical data

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 5. ERC-20 Token Audit Prompt

```
Audit the following ERC-20 token implementation for correctness and security.

Verify:
1. Full compliance with EIP-20 specification (correct return values, events, edge cases)
2. Correct handling of the approval race condition
3. Safe handling of fee-on-transfer if applicable
4. Overflow/underflow safety
5. Mint/burn access control
6. Blacklist or pausable functionality risks
7. Upgrade proxy risks (if upgradeable)
8. Any deviation from OpenZeppelin's reference implementation and its security implications

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 6. DeFi Protocol Review Prompt

```
You are a DeFi security specialist. Audit the following protocol for financial and technical vulnerabilities.

Analyze for:
1. Price oracle manipulation (TWAP vs spot price, flash loan attacks on oracles)
2. Flash loan attack vectors (can atomic borrow+exploit+repay drain the protocol?)
3. Liquidation logic correctness (under-collateralization edge cases)
4. Rounding errors that accumulate to exploitable amounts
5. Front-running of deposits, withdrawals, or liquidations
6. Sandwich attack exposure on AMM interactions
7. Slippage parameter enforcement
8. Emergency pause / circuit breaker presence

For each finding, provide a proof-of-concept attack scenario and the recommended mitigation.

Protocol contracts:
\`\`\`solidity
[PASTE CONTRACTS HERE]
\`\`\`
```

---

## 7. Upgradeable Contract Review Prompt

```
Audit the following upgradeable smart contract system for upgrade-related vulnerabilities.

Check:
1. Storage layout conflicts between implementation versions (list all storage slots)
2. Proper use of initializer (no constructor logic, `initializer` modifier present)
3. Initialization front-running risk (is `initialize` called atomically with deployment?)
4. UUPS vs Transparent Proxy pattern correctness
5. Access control on upgrade functions (`upgradeTo`, `upgradeToAndCall`)
6. Self-destruct risk in implementation contracts (for UUPS)
7. Delegatecall safety (no ETH-stealing delegatecall vulnerability)

Contracts (proxy + implementation):
\`\`\`solidity
[PASTE CONTRACTS HERE]
\`\`\`
```

---

## 8. Audit Report Drafting Prompt

After identifying vulnerabilities manually or with prior prompts, use this to draft a structured report:

```
Based on the following findings from a smart contract audit, write a professional security audit report section.

For each finding, format it as:
---
**[SEVERITY] Finding Title**

**Description:** Clear explanation of the vulnerability.

**Impact:** What an attacker can achieve.

**Proof of Concept:**
\`\`\`solidity
// Minimal attack contract or step-by-step scenario
\`\`\`

**Recommendation:**
\`\`\`solidity
// Fixed code
\`\`\`

**References:** Relevant SWC IDs, EIP links, or CVEs.
---

Findings:
[PASTE YOUR RAW FINDINGS HERE]
```

---

## 9. Audit Checklist Generation Prompt

```
Generate a comprehensive smart contract audit checklist for a [TYPE: DEX / Lending / NFT / Staking / Bridge] protocol built on Ethereum.

The checklist should be organized by category:
- Access Control
- Arithmetic & Logic
- External Calls & Reentrancy
- Token Handling (ERC-20 / ERC-721 edge cases)
- Oracle & Price Feeds
- Upgradability
- Gas & DoS
- Business Logic

For each item, include: the check description, how to test it, and which tool can help (Slither, Echidna, Foundry invariant tests, manual review).
```

---

## 10. Tips for Using AI in Audits

| Do | Don't |
|----|-------|
| Provide full contract code (with imports where possible) | Share only snippets — context matters |
| Ask about one vulnerability class at a time for depth | Ask "find all bugs" — too broad, misses nuance |
| Verify every finding on a local fork | Trust AI output without manual verification |
| Use AI to explain unfamiliar patterns | Skip professional audit for high-value contracts |
| Ask AI to generate PoC exploit code for verification | Rely on severity ratings without checking impact |

---

## References

- [SWC Registry](https://swcregistry.io/)
- [Consensys Smart Contract Best Practices](https://consensys.github.io/smart-contract-best-practices/)
- [OpenZeppelin Defender (automated monitoring)](https://www.openzeppelin.com/defender)
- [Slither – Static Analyzer](https://github.com/crytic/slither)
- [Echidna – Fuzzer](https://github.com/crytic/echidna)
- [Foundry – Testing Framework with Invariant Tests](https://book.getfoundry.sh/)
