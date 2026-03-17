# EIP-20 (ERC-20): Fungible Token Standard

> **Date:** 2024-01-01  
> **Tags:** eip-20, erc-20, token, fungible, ethereum, standard

## Introduction

**EIP-20** (also known as ERC-20) is the most widely adopted token standard on Ethereum, proposed by Fabian Vogelsteller and Vitalik Buterin in November 2015. It defines a uniform interface for fungible tokens, enabling wallets, DEXs, and other protocols to interact with any token without needing to know its implementation details.

---

## 1. Motivation

Before ERC-20, every token project implemented its own unique interface. Exchanges had to write custom integration code for each token. EIP-20 solved this by specifying a minimal, composable interface that the entire ecosystem could build on.

---

## 2. The Interface

```solidity
interface IERC20 {
    // ── State-Changing ────────────────────────────────────────────────────────

    /// @notice Transfer `value` tokens from caller to `to`.
    function transfer(address to, uint256 value) external returns (bool);

    /// @notice Transfer `value` tokens from `from` to `to` using the allowance mechanism.
    function transferFrom(address from, address to, uint256 value) external returns (bool);

    /// @notice Approve `spender` to spend up to `value` tokens on behalf of caller.
    function approve(address spender, uint256 value) external returns (bool);

    // ── View ──────────────────────────────────────────────────────────────────

    /// @notice Total token supply.
    function totalSupply() external view returns (uint256);

    /// @notice Token balance of `account`.
    function balanceOf(address account) external view returns (uint256);

    /// @notice Amount `spender` is allowed to spend on behalf of `owner`.
    function allowance(address owner, address spender) external view returns (uint256);

    // ── Events ────────────────────────────────────────────────────────────────

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

### 2.1 Key Rules

- `transfer` and `transferFrom` **must** emit a `Transfer` event.
- `approve` **must** emit an `Approval` event.
- `transfer` from/to the zero address (`0x000...000`) is treated as minting/burning by convention.
- Functions **should** return `true` on success. Many early implementations did not return a value — use OpenZeppelin's `SafeERC20` wrapper to handle such tokens.

---

## 3. The Allowance Mechanism

The allowance flow enables third-party contracts (DEXs, lending protocols, etc.) to spend tokens on behalf of a user:

```
User                      DEX Router
 │                             │
 │── approve(router, 100) ────►│   router.allowance[user] = 100
 │                             │
 │── swap(token, 100, ...) ───►│
 │                             │── transferFrom(user, pool, 100) ──► Pool
 │                             │   deducts from allowance
```

### 3.1 Approval Race Condition

Changing an allowance from a non-zero value to another non-zero value is vulnerable to a front-running attack:

1. Alice approves Bob for 100.
2. Alice sends a tx to change allowance to 50.
3. Bob sees the pending tx, front-runs `transferFrom` spending 100, then after Alice's tx, spends 50.
4. Bob spent 150 total, not 50.

**Mitigations:**
- Set allowance to 0 first, then to the new value.
- Use `increaseAllowance` / `decreaseAllowance` helper functions (non-standard but widely adopted).
- Use **EIP-2612 Permit** (see §5) to avoid on-chain approvals entirely.

---

## 4. Optional Extension Functions

The EIP defines three optional metadata functions:

```solidity
function name()     external view returns (string memory); // e.g., "USD Coin"
function symbol()   external view returns (string memory); // e.g., "USDC"
function decimals() external view returns (uint8);         // typically 18; USDC uses 6
```

> `decimals()` is critical: `1 USDC = 1_000_000` units (6 decimals), while `1 ETH = 1e18 wei`. Always use token decimals when displaying human-readable amounts.

---

## 5. EIP-2612: Permit (Gasless Approvals)

EIP-2612 extends ERC-20 with a `permit` function that accepts an EIP-712 typed signature instead of requiring an on-chain approval transaction. This enables **gasless token approvals** (relayer pays gas; user signs off-chain).

```solidity
function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v, bytes32 r, bytes32 s
) external;
```

Tokens like USDC, DAI, and UNI support Permit. It is the foundation of "swap without prior approval" UX in modern DEX aggregators.

---

## 6. Common Implementation Patterns

### 6.1 OpenZeppelin ERC20

The de-facto standard implementation. Inheriting from it is the safest starting point:

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("My Token", "MTK") {
        _mint(msg.sender, initialSupply);
    }
}
```

### 6.2 Fee-On-Transfer Tokens

Some tokens deduct a fee during `transfer`, so the recipient receives less than `value`. Protocols that interact with arbitrary ERC-20 tokens **must** measure the actual balance change:

```solidity
uint256 before = token.balanceOf(address(this));
token.transferFrom(msg.sender, address(this), amount);
uint256 received = token.balanceOf(address(this)) - before;
// use `received`, not `amount`
```

### 6.3 Rebasing Tokens

Tokens like stETH adjust all balances simultaneously via a global multiplier. Their `balanceOf` return value changes between blocks. Store **shares**, not raw balances, when integrating rebasing tokens.

---

## 7. Security Considerations

| Risk | Description | Mitigation |
|------|-------------|------------|
| Missing return value | Old tokens (USDT) don't return `bool` | Use `SafeERC20.safeTransfer` |
| Fee-on-transfer | Received < sent | Check actual balance delta |
| Reentrancy on `transfer` | Malicious `ERC777` hooks | Follow CEI pattern |
| Approval front-running | Double-spend via race condition | Zero-then-set or use Permit |
| Infinite approval | Approvals never expire | Scope approvals to exact amounts where possible |

---

## 8. Ecosystem Impact

ERC-20 underpins virtually all DeFi:
- **DEXs:** Uniswap, Curve swap ERC-20 pairs.
- **Lending:** Aave, Compound accept ERC-20 collateral.
- **Stablecoins:** USDC, USDT, DAI are ERC-20.
- **Governance:** UNI, COMP, AAVE tokens are ERC-20 with vote delegation.

---

## References

- [EIP-20 Specification](https://eips.ethereum.org/EIPS/eip-20)
- [EIP-2612: Permit](https://eips.ethereum.org/EIPS/eip-2612)
- [OpenZeppelin ERC20 Source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)
- [SafeERC20](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20)
