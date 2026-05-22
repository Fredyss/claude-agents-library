---
name: solidity-dapp
description: Write, review, test, and reason about Solidity smart contracts and full-stack Ethereum/EVM dApps. Use this skill whenever the user is writing or reviewing Solidity (.sol) code, building or debugging a smart contract (token, NFT, vault, DAO, escrow, auction, staking, etc.), asks about Solidity language features (types, mappings, modifiers, storage vs memory, custom errors, inheritance), wants ERC standards (ERC-20, ERC-721, ERC-1155) or OpenZeppelin, needs help with smart-contract security (reentrancy, access control, overflow), works with Foundry/Hardhat/Remix, deploys or verifies a contract, optimizes gas, or connects a contract to a frontend with ethers.js/viem/wagmi — even if they don't say "Solidity" or "dApp" explicitly. Prefer this skill for anything involving EVM smart contracts or web3 frontends.
---

# Solidity & dApp Development

A practical toolkit for building EVM smart contracts and the dApps around them, current to Solidity **v0.8.35** (latest as of this writing). Smart contracts are unusual software: they are immutable once deployed, publicly visible, hold real value, and run in an adversarial environment where any bug is an open invitation. So this skill is **security-first** — correctness and safety outrank cleverness and even gas savings.

Use it to write new contracts, review existing ones, debug, test, deploy, optimize gas, or wire a contract up to a frontend.

## Non-negotiable defaults

Apply these unless the user has a specific reason to deviate. They reflect current (0.8.x) best practice, not legacy tutorials.

- **Pin a recent compiler.** Start files with `// SPDX-License-Identifier: MIT` then `pragma solidity ^0.8.20;` (or newer). A license identifier and a pinned pragma are expected on every file.
- **Arithmetic is checked by default.** Since 0.8.0, `+`, `-`, `*` revert on overflow/underflow. Only wrap math in `unchecked { }` when you've *proven* it can't overflow and need the gas — and say why.
- **Don't reinvent standards.** Use audited **OpenZeppelin** contracts for ERC-20/721/1155, `Ownable`, `AccessControl`, `ReentrancyGuard`, `SafeERC20`, pausing, and proxies. Hand-rolling these is a leading source of bugs.
- **Prefer `call` for sending ETH, not `transfer`/`send`.** `transfer`/`send` forward a fixed 2300 gas and break with smart-contract wallets; they're also being deprecated (warned since 0.8.31). Use `(bool ok, ) = recipient.call{value: amount}(""); require(ok);` — and guard against reentrancy.
- **Use custom errors, not `require` strings.** `error Unauthorized();` + `revert Unauthorized();` is far cheaper than `require(cond, "Unauthorized")` and clearer. (Available since 0.8.4.)
- **Follow Checks-Effects-Interactions** in every function that touches external addresses (validate → update your own state → *then* make external calls).

## A minimal, well-formed contract

This is the shape every contract should start from — explicit visibility, events on state changes, custom errors, CEI ordering:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Vault {
    mapping(address => uint256) private balances;

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    error InsufficientBalance(uint256 requested, uint256 available);

    function deposit() external payable {
        balances[msg.sender] += msg.value;     // checked arithmetic
        emit Deposited(msg.sender, msg.value);
    }

    function withdraw(uint256 amount) external {
        uint256 bal = balances[msg.sender];
        if (amount > bal) revert InsufficientBalance(amount, bal);
        balances[msg.sender] = bal - amount;   // effects BEFORE interaction (CEI)
        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok, "transfer failed");
        emit Withdrawn(msg.sender, amount);
    }
}
```

## The development workflow

1. **Specify the contract's job and trust model first.** What state does it hold? Who is allowed to call what? What value flows through it, and what must *never* happen (the invariants)? Security follows from a clear trust model — write it down before coding.
2. **Model state and access control.** Choose data structures (mappings vs arrays — see `language.md`), and decide the access-control scheme (`Ownable`, roles via `AccessControl`) up front.
3. **Write the contract** following the defaults above. Lean on OpenZeppelin. Keep functions small; mark visibility explicitly; emit events on every meaningful state change. See `language.md` and `contract-patterns.md`.
4. **Threat-model and self-audit** against the checklist in `security.md` *before* testing — reentrancy, access control, arithmetic, external-call assumptions, oracle/randomness, front-running.
5. **Test exhaustively, including adversarial cases.** Unit tests for the happy path, then fuzz/invariant tests and explicit attacker scenarios. Smart contracts can't be patched after deploy. See `tooling-and-testing.md`.
6. **Optimize gas last** — only after it's correct and tested, and never at the expense of safety or readability. See `tooling-and-testing.md`.
7. **Deploy, verify, and integrate.** Deploy via script, verify source on the explorer, then connect the frontend. See `dapp-integration.md`.

## Reference routing

Read only the files a given task touches; each is self-contained.

| If the task involves… | Read |
|---|---|
| Solidity syntax — types, mappings, structs, functions, visibility, modifiers, storage/memory/calldata, events, errors, inheritance, interfaces, 0.8 semantics | `references/language.md` |
| Reusable patterns and standards — ERC-20/721/1155, OpenZeppelin, access control, pausing, pull-payments, factories, upgradeable proxies | `references/contract-patterns.md` |
| Security — reentrancy, access-control flaws, arithmetic, `tx.origin`, randomness, front-running/MEV, oracle manipulation, the audit checklist | `references/security.md` |
| Tooling and testing — Foundry, Hardhat, Remix, unit/fuzz/invariant tests, deployment scripts, verification, gas optimization | `references/tooling-and-testing.md` |
| Frontend / full-stack — ethers.js, viem, wagmi, ABIs, providers/signers, wallet connection, reading/writing contracts, listening to events | `references/dapp-integration.md` |

## Operating principles

- **Assume the caller is hostile.** Every external function is a public attack surface. Validate inputs, check authorization, and never trust external contracts to behave.
- **Immutable means get it right the first time.** There's no hotfix. This justifies the heavy emphasis on review and testing.
- **Favor clarity over cleverness.** Auditors (and your future self) must be able to read it. A subtle gas trick that obscures a reentrancy path is a bad trade.
- **Tie every recommendation to the threat model.** "Use a reentrancy guard here because this function makes an external call before all state is settled" beats "add a guard for safety."
