# Solidity Language Reference

Core language mechanics, current to the 0.8.x series. The recurring theme: Solidity exposes the EVM's cost and storage model directly, so seemingly small choices (where data lives, what visibility a function has) have real consequences for gas and safety.

## Contents
- File and contract structure
- Value vs reference types
- Data location: storage, memory, calldata
- Mappings, arrays, structs, enums
- Functions: visibility, state mutability, modifiers
- Special functions: constructor, receive, fallback
- Events
- Errors and reverts
- Inheritance and interfaces
- 0.8 semantics worth knowing

## File and contract structure

Every file starts with an SPDX license comment and a version pragma:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;
```

A `contract` groups state variables, functions, events, errors, modifiers, and type definitions. Other top-level constructs: `interface` (function signatures only, no implementation/state), `library` (stateless reusable code, often attached to types with `using ... for`), and free functions/constants/errors declared outside any contract.

## Value vs reference types

**Value types** are copied on assignment and when passed to functions:
- `uint`/`int` (sized: `uint8` … `uint256`; `uint` = `uint256`). Default to `uint256` unless packing storage.
- `bool`, `address` (and `address payable`, which can receive ETH), `bytes1`…`bytes32` (fixed), `enum`.
- There are **no floats** — represent decimals as integers with a scaling factor (e.g. token amounts use 18 decimals).

**Reference types** point to data and require an explicit data location:
- `string`, `bytes` (dynamic byte array), arrays (`T[]` dynamic, `T[N]` fixed), `struct`, `mapping`.

`address` exposes `.balance`, and members for sending value/calling: `.call`, `.delegatecall`, `.staticcall`. Prefer `.call{value: x}("")` for sending ETH (see security notes on avoiding `transfer`/`send`).

## Data location: storage, memory, calldata

This is the concept newcomers stumble on most, and it controls both cost and behavior:
- **storage** — persistent on-chain state; the contract's permanent variables. Most expensive to write. A local `storage` reference *aliases* the original (mutating it mutates state).
- **memory** — temporary, mutable, wiped after the call. Used for working copies and most function locals of reference type.
- **calldata** — read-only, non-persistent; where external function arguments live. Cheapest for input you don't modify; prefer `calldata` over `memory` for external function reference-type params.

Reference-type parameters and locals must declare a location. Assigning a `storage` array to a `memory` variable copies it; assigning to a `storage` local creates an alias — a frequent source of bugs.

## Mappings, arrays, structs, enums

- **mapping(K => V)** — a hash table; the workhorse for "balance per address," "owner of token id," etc. Cannot be iterated, has no length, and can't live in memory. If you need enumeration, maintain a separate array of keys alongside it.
- **arrays** — dynamic (`uint[]`) or fixed (`uint[10]`). Beware unbounded loops over arrays that grow with user input: a loop that's cheap today can exceed the block gas limit later and brick the function. Prefer mappings or the pull-payment pattern over iterating user-controlled arrays.
- **structs** — group related fields. Mind storage packing: variables sharing a 32-byte slot (e.g. several `uint128`/`bool`/`address`) cost less; ordering fields by size can save a slot.
- **enums** — named integer constants for state machines.

## Functions: visibility, state mutability, modifiers

**Visibility (always specify it):**
- `external` — callable only from outside the contract; cheapest for external calls; args can be `calldata`.
- `public` — callable internally and externally; auto-generates a getter for `public` state vars.
- `internal` — this contract and derived contracts (default for state variables).
- `private` — this contract only. Note "private" hides it from other contracts, **not** from observers — all on-chain data is publicly readable.

**State mutability:**
- `view` — reads state, doesn't modify it.
- `pure` — neither reads nor modifies state.
- `payable` — can receive ETH (`msg.value`). Functions that aren't `payable` reject ETH.
- (no keyword) — may modify state.

**Modifiers** factor out pre/post-condition checks (e.g. an `onlyOwner` guard). They run around the function body where `_;` marks the insertion point. Keep them simple and side-effect-light; complex logic belongs in functions.

Key globals inside functions: `msg.sender` (immediate caller), `msg.value` (ETH sent), `block.timestamp`, `block.number`, `address(this)`. Do **not** use `tx.origin` for authorization (see `security.md`).

## Special functions

- **constructor** — runs once at deployment to initialize state; not part of the deployed code.
- **receive() external payable** — invoked on plain ETH transfers with empty calldata.
- **fallback() external [payable]** — invoked when no function matches the call (or for ETH transfers if there's no `receive`). Common in proxy patterns.

## Events

Events write to the cheap, append-only log; they're the primary way off-chain apps (and indexers) observe contract activity. Contracts cannot read their own past events.

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);
emit Transfer(msg.sender, to, amount);
```

Mark up to three params `indexed` to make them filterable/searchable by clients. Emit an event on every meaningful state change — frontends depend on them.

## Errors and reverts

A revert undoes all state changes in the transaction and refunds remaining gas. Three forms:
- **Custom errors** (preferred, since 0.8.4): `error InsufficientBalance(uint256 needed, uint256 have);` then `revert InsufficientBalance(needed, have);`. Cheapest, and can carry structured data.
- **`require(condition, "msg")`** — validate inputs/preconditions. Still common; custom errors are cheaper.
- **`revert("msg")`** — unconditional bail-out.
- **`assert(condition)`** — for invariants that should *never* be false; a failing assert signals a bug, not bad input.

## Inheritance and interfaces

- `contract Child is Parent { ... }`; multiple inheritance is allowed and linearized C3-style — list base contracts from most base-like to most derived.
- Override requires `virtual` on the base function and `override` on the derived one (enforced since 0.6.0). `super.foo()` calls the next implementation in the chain.
- **interfaces** declare external function signatures with no bodies or state — use them to call other contracts in a typed way (e.g. `IERC20(token).transfer(to, amt)`).
- **libraries** hold reusable stateless logic; attach with `using SafeERC20 for IERC20;` so you can call `token.safeTransfer(...)`.

## 0.8 semantics worth knowing

- **Checked arithmetic by default** (since 0.8.0): overflow/underflow revert. Use `unchecked { }` only with a proven-safe justification.
- **ABI coder v2 is default** (since 0.8.0) — supports nested/dynamic types and does more input validation.
- **`send`/`transfer` deprecation**: warned since 0.8.31; prefer `call` with a reentrancy guard.
- **Deprecations heading toward 0.9.0**: identifiers like `at`, `error`, `layout`, `leave`, `super`, `this`, `transient` now emit warnings as they're slated to become keywords — avoid using them as plain names.
- Always develop against the latest released compiler; only the newest line receives security fixes.
