# Solidity Code Generation AI Prompts

> **Date:** 2024-01-01  
> **Tags:** ai, prompts, solidity, code generation, llm, development

## Introduction

This article provides curated prompts for using AI assistants to generate, explain, refactor, and review Solidity code. Each prompt is designed to produce high-quality, production-ready output and includes context tips to get the best results.

---

## 1. ERC-20 Token Generation Prompt

```
Write a production-ready ERC-20 token in Solidity 0.8.x using OpenZeppelin Contracts v5.

Requirements:
- Token name: [NAME], symbol: [SYMBOL], decimals: 18
- Initial supply minted to deployer: [AMOUNT]
- Mintable by addresses with MINTER_ROLE
- Burnable by token holders
- Pausable by addresses with PAUSER_ROLE
- Upgradeable via UUPS proxy pattern
- Include NatSpec documentation on all public functions
- Include a comprehensive Foundry test file covering: minting, burning, pausing, role management, and upgrade

Use OpenZeppelin Contracts Upgradeable library. Follow the CEI pattern. Use custom errors.
```

---

## 2. ERC-721 NFT Generation Prompt

```
Write a production-ready ERC-721 NFT contract in Solidity 0.8.x using OpenZeppelin Contracts v5.

Requirements:
- Collection name: [NAME], symbol: [SYMBOL]
- Maximum supply: [MAX_SUPPLY]
- Public mint price: [PRICE] ETH, max [N] per wallet
- Reveal mechanic: baseURI is mutable by owner, default points to pre-reveal image
- Provenance hash committed on-chain before reveal
- EIP-2981 royalty support: [X]% to deployer
- Withdraw function for owner to pull ETH
- Use ERC721A (Azuki) for efficient batch minting
- Include NatSpec and a Foundry test file covering: mint, batch mint, wallet limit, reveal, royalties, withdrawal

Use custom errors. Emit events for all state changes.
```

---

## 3. Multi-Sig Wallet Generation Prompt

```
Write a minimal multi-signature wallet contract in Solidity 0.8.x.

Requirements:
- Owners set at deployment (array of addresses, immutable)
- Configurable confirmation threshold (e.g., 2-of-3)
- Submit transaction: any owner can propose a call (to, value, data)
- Confirm transaction: any owner can confirm a pending proposal
- Execute transaction: callable by any owner once threshold is met
- Revoke confirmation: owner can withdraw their confirmation before execution
- Emit events for: Submission, Confirmation, Revocation, Execution, ExecutionFailed, Deposit

Security requirements:
- Prevent duplicate owner addresses
- Prevent non-owner access
- Prevent double confirmation from same owner
- Follow CEI pattern for execution

Include NatSpec and a Foundry test suite.
```

---

## 4. Staking Contract Generation Prompt

```
Write a token staking contract in Solidity 0.8.x.

Requirements:
- Users stake [STAKE_TOKEN] ERC-20 and earn [REWARD_TOKEN] ERC-20
- Reward rate: [RATE] reward tokens per second per staked token (or per block)
- Rewards are accumulated per-user using a "reward per token stored" pattern (Synthetix-style)
- Functions: stake(amount), withdraw(amount), claimReward(), exit()
- Owner can set reward rate and fund the reward pool
- No lock-up period (immediate withdrawal allowed)
- Emit events for Staked, Withdrawn, RewardPaid, RewardAdded

Use the Synthetix StakingRewards formula to avoid per-user loops. Include NatSpec and a Foundry test covering: stake, withdraw, reward accrual, and multiple users.
```

---

## 5. Contract Explanation Prompt

```
You are an expert Solidity developer and educator.

Explain the following smart contract in detail:
1. What is the contract's purpose and high-level behavior?
2. Explain each state variable: type, purpose, storage implications.
3. Walk through each function: inputs, logic, outputs, events emitted, gas cost considerations.
4. Identify any non-obvious patterns or design choices (e.g., bitmaps, assembly, reentrancy guards).
5. Note any potential risks or trade-offs in the current design.

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 6. Code Refactoring Prompt

```
Refactor the following Solidity contract to improve:
1. **Security:** Apply the CEI pattern, add reentrancy guards where needed, use custom errors.
2. **Gas efficiency:** Pack storage variables, cache SLOADs, use unchecked for safe arithmetic, prefer calldata over memory.
3. **Readability:** Add NatSpec comments, rename unclear variables, extract complex logic into internal functions.
4. **Standards compliance:** Ensure full EIP-20 / EIP-721 compliance if applicable.

Preserve all existing functionality exactly. For each change, add an inline comment explaining the optimization or fix.

Original contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 7. Unit Test Generation Prompt (Foundry)

```
Write a comprehensive Foundry test suite in Solidity for the following smart contract.

Test requirements:
- Cover all public and external functions
- Test both happy paths and revert cases (use vm.expectRevert)
- Use fuzzing (uint256 amount, address user) where appropriate
- Test event emissions (vm.expectEmit)
- Test access control (non-owner should revert)
- Use setUp() to deploy the contract with required dependencies
- Name tests: test_<Function>_<Scenario> (e.g., test_Withdraw_RevertsWhenAmountExceedsBalance)

Contract to test:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 8. ABI & Interface Generation Prompt

```
Given the following Solidity contract, generate:
1. The Solidity interface (IERC-style) with NatSpec for all external functions and events.
2. The TypeScript/ethers.js v6 typed contract class using typechain-compatible patterns.
3. A JavaScript snippet showing how to:
   - Connect to the contract on [NETWORK]
   - Call a read function: [FUNCTION_NAME]
   - Send a write transaction: [FUNCTION_NAME]
   - Listen to an event: [EVENT_NAME]

Contract:
\`\`\`solidity
[PASTE CONTRACT HERE]
\`\`\`
```

---

## 9. Deployment Script Generation Prompt

```
Write a Foundry deployment script (Script.sol) for the following contract system.

Deployment order:
1. [CONTRACT_A] – constructor args: [ARGS]
2. [CONTRACT_B] – depends on CONTRACT_A address
3. [Initialize or configure steps]

Requirements:
- Use vm.startBroadcast() / vm.stopBroadcast()
- Read deployer private key from environment (vm.envUint("PRIVATE_KEY"))
- Log deployed addresses with console.log
- Write addresses to a JSON file using vm.writeJson for use by frontend
- Support both local Anvil and mainnet/testnet deployment (use block.chainid checks if needed)

Contracts:
\`\`\`solidity
[PASTE CONTRACTS HERE]
\`\`\`
```

---

## 10. Prompt Engineering Tips for Solidity

| Tip | Example |
|-----|---------|
| **Specify Solidity version** | "Use Solidity 0.8.24" |
| **Specify library versions** | "Use OpenZeppelin Contracts v5.0" |
| **Request test framework** | "Write tests in Foundry / Hardhat / Truffle" |
| **Specify naming conventions** | "Follow OpenZeppelin naming: `_` prefix for internal state" |
| **Ask for NatSpec** | "Add @notice, @param, @return NatSpec on all functions" |
| **Request custom errors** | "Use custom errors instead of require strings" |
| **Provide context** | "This is a yield aggregator for Aave V3" |
| **Iterate on output** | "Now add pausability to the contract you just generated" |
| **Ask for alternatives** | "Show me two different implementations: one using mappings, one using an array" |

---

## References

- [Foundry Book](https://book.getfoundry.sh/)
- [OpenZeppelin Contracts v5 Docs](https://docs.openzeppelin.com/contracts/5.x/)
- [Solidity Documentation](https://docs.soliditylang.org/)
- [Synthetix StakingRewards Pattern](https://github.com/Synthetixio/synthetix/blob/develop/contracts/StakingRewards.sol)
- [NatSpec Format](https://docs.soliditylang.org/en/latest/natspec-format.html)
