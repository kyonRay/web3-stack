# Solidity Best Practices & Security Patterns

> **Date:** 2024-01-01  
> **Tags:** solidity, security, smart contracts, patterns, auditing

## Introduction

Writing secure Solidity code requires discipline beyond syntax knowledge. The immutable and public nature of deployed contracts means that mistakes are costly and permanent. This article documents the most critical security patterns and coding standards to follow.

---

## 1. Checks-Effects-Interactions (CEI) Pattern

The single most important pattern for preventing **reentrancy** attacks.

**Rule:** Always perform all state changes before making external calls.

```solidity
// VULNERABLE – Reentrancy possible
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool success, ) = msg.sender.call{value: amount}(""); // external call FIRST
    require(success);
    balances[msg.sender] -= amount; // state change AFTER
}

// SAFE – CEI pattern applied
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount); // Check
    balances[msg.sender] -= amount;           // Effect
    (bool success, ) = msg.sender.call{value: amount}(""); // Interaction
    require(success);
}
```

If you must make multiple external calls, also consider a **Reentrancy Guard**:

```solidity
uint256 private _status = 1; // 1 = not entered

modifier nonReentrant() {
    require(_status == 1, "ReentrancyGuard: reentrant call");
    _status = 2;
    _;
    _status = 1;
}
```

OpenZeppelin's `ReentrancyGuard` implements this pattern and should be preferred in production.

---

## 2. Access Control

### 2.1 Use `Ownable` or Role-Based Access

Always restrict privileged functions. Avoid rolling your own access control; use OpenZeppelin's `Ownable` or `AccessControl`.

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";

contract Vault is Ownable {
    function sweep(address token) external onlyOwner { ... }
}
```

For multi-role systems:

```solidity
import "@openzeppelin/contracts/access/AccessControl.sol";

bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

// Grant: grantRole(PAUSER_ROLE, address)
// Check: hasRole(PAUSER_ROLE, msg.sender)
```

### 2.2 Two-Step Ownership Transfer

Transferring ownership in one step risks transferring to a wrong address permanently. Use `Ownable2Step`:

```solidity
import "@openzeppelin/contracts/access/Ownable2Step.sol";
// pendingOwner must call acceptOwnership() to finalize
```

---

## 3. Integer Overflow and Underflow

Solidity ≥ 0.8.0 includes built-in overflow/underflow checks. For older versions, use OpenZeppelin's `SafeMath`. For ≥ 0.8.0, be cautious of `unchecked` blocks — only use them when you have mathematically proven safety.

```solidity
// Safe in Solidity 0.8+ – reverts on overflow
uint256 result = a + b;

// Dangerous – bypass overflow check only when certain
unchecked {
    uint256 result = a + b; // must be provably safe
}
```

---

## 4. `msg.value` in Loops

Never use `msg.value` inside a loop. Each iteration refers to the same original `msg.value`, but funds are only sent once.

```solidity
// VULNERABLE
function batchDeposit(address[] calldata recipients) external payable {
    for (uint i = 0; i < recipients.length; i++) {
        deposit(recipients[i], msg.value); // same value every iteration!
    }
}
```

---

## 5. Use `calldata` for Read-Only External Inputs

Function parameters from external callers that are not modified should use `calldata` instead of `memory` to save gas.

```solidity
// Prefer calldata for external functions
function processItems(uint256[] calldata ids) external view returns (uint256) { ... }
// Use memory for internal/public
function processItems(uint256[] memory ids) internal view returns (uint256) { ... }
```

---

## 6. Front-Running Mitigation

Public mempool transactions can be seen and front-run. Strategies:

| Strategy | Description |
|----------|-------------|
| **Commit-reveal** | Users submit a hash of their action first, reveal later |
| **Slippage tolerance** | Allow callers to set min/max acceptable values |
| **Private mempools** | Use Flashbots `eth_sendPrivateTransaction` for sensitive txs |
| **Batch auctions** | Aggregate orders and settle in one block (CoW Protocol) |

---

## 7. Storage Layout and Upgradeable Contracts

For upgradeable contracts (Transparent Proxy, UUPS), **storage layout must not change** between upgrades:

```solidity
// V1
contract StorageV1 {
    uint256 public value;  // slot 0
    address public owner;  // slot 1
}

// V2 – CORRECT: only append new variables
contract StorageV2 is StorageV1 {
    uint256 public extra; // slot 2 (new)
}

// V2 – WRONG: inserting variable shifts layout
contract StorageV2Bad {
    uint256 public newVar; // slot 0 – CLASHES with value!
    uint256 public value;  // slot 1 – CLASHES with owner!
    address public owner;  // slot 2
}
```

Use OpenZeppelin's `Upgrades` plugin (`hardhat-upgrades`) to validate storage layout safety before upgrading.

---

## 8. Error Handling

Prefer `custom errors` over `require` strings to save gas and improve debuggability:

```solidity
// Costs more gas – string stored in bytecode
require(balance >= amount, "Insufficient balance");

// Preferred – cheaper, selectable in catch blocks
error InsufficientBalance(uint256 available, uint256 required);

if (balance < amount) revert InsufficientBalance(balance, amount);
```

---

## 9. Events and Indexing

Emit events for all state-changing operations to enable off-chain indexing. Index parameters that will be used as filters:

```solidity
event Transfer(
    address indexed from,  // indexed for filtering
    address indexed to,
    uint256 value          // not indexed – just value
);
```

---

## 10. Code Review Checklist

Before deploying any contract, verify:

- [ ] All external calls follow CEI or use `nonReentrant`
- [ ] Every privileged function has access control
- [ ] No `tx.origin` for authorization (use `msg.sender`)
- [ ] No hardcoded addresses (use constructor parameters or governance)
- [ ] All arithmetic in `unchecked` blocks is mathematically proven safe
- [ ] Storage layout documented and verified for upgradeable contracts
- [ ] Contract size < 24 KB (Spurious Dragon limit)
- [ ] Tests cover both happy paths and edge cases
- [ ] Third-party audit obtained for high-value contracts

---

## References

- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
- [SWC Registry – Smart Contract Weakness Classification](https://swcregistry.io/)
- [Consensys Smart Contract Best Practices](https://consensys.github.io/smart-contract-best-practices/)
- [Solidity Docs – Security Considerations](https://docs.soliditylang.org/en/latest/security-considerations.html)
