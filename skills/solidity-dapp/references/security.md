# Smart Contract Security

The single most important reference here. Smart contracts are immutable, public, and hold value, so they are attacked relentlessly and bugs are unrecoverable. Treat every external function as a hostile entry point and assume any contract you call may try to exploit you.

## Contents
- The mindset
- Reentrancy
- Access control failures
- Arithmetic
- Unsafe external calls and ETH handling
- tx.origin authentication
- Bad randomness
- Front-running / MEV
- Oracle and price manipulation
- Denial of service
- Pre-deploy audit checklist

## The mindset

Security review is not a final step; it's how you write each function. For every external function ask: *who can call this, what do they control, what's the worst input, and what happens if a contract I call reenters or reverts?* The patterns below are the recurring answers.

## Reentrancy

The canonical exploit. If a function makes an external call (sending ETH, calling another contract) **before** it finishes updating its own state, the called contract can call back in and exploit the stale state — e.g. withdraw repeatedly before the balance is zeroed (the DAO hack).

Defenses, in order of preference:
1. **Checks-Effects-Interactions (CEI)**: validate inputs (checks) → update your own state (effects) → make external calls (interactions) *last*. Most reentrancy vanishes if state is settled before the external call.
2. **Reentrancy guard**: OpenZeppelin's `nonReentrant` modifier on functions that can't be made CEI-clean.
3. Be aware of **cross-function** and **read-only** reentrancy: a guard on one function doesn't protect another that shares state, and view functions can return mid-update state to other contracts.

## Access control failures

The most common root cause of real losses. Symptoms: a privileged function (mint, withdraw, upgrade, set-owner, pause) missing its `onlyOwner`/role check; an `initialize()` left unprotected so anyone can claim ownership; default-`public` functions that should be restricted. Mitigation: explicitly guard every privileged function, use `Ownable2Step`/`AccessControl`, put high-value control behind a multisig/timelock, and test that unauthorized callers are rejected.

## Arithmetic

Since Solidity 0.8.0, `+`/`-`/`*` **revert on overflow/underflow by default**, which kills the classic overflow exploits — *as long as you don't wrap in `unchecked { }`*. Rules:
- Only use `unchecked` when you've proven the operation cannot overflow (e.g. a counter bounded by an array length) and you need the gas; comment why.
- Division truncates toward zero — beware precision loss; multiply before dividing, and scale with fixed-point (e.g. 1e18) where needed.
- Casting down (`uint256` → `uint128`) silently truncates; use OpenZeppelin `SafeCast`.

## Unsafe external calls and ETH handling

- **Send ETH with `call`, not `transfer`/`send`.** `transfer`/`send` cap gas at 2300 (breaks recipients that are smart-contract wallets or have non-trivial `receive`) and are being deprecated. Use `(bool ok, ) = to.call{value: amount}(""); require(ok);` — and protect it with CEI/`nonReentrant`.
- **Always check return values.** A low-level `call`/`delegatecall` returns a success bool that must be checked; a failed call does *not* auto-revert. Many ERC-20s return non-standard values — use `SafeERC20`.
- **`delegatecall` is dangerous**: it runs external code in *your* storage context. Only delegatecall trusted code; it's the heart of proxy patterns and also of several catastrophic hacks.
- Don't assume `address(this).balance` only changes through your functions — ETH can be force-sent (e.g. via `selfdestruct`), so never use exact balance equality as an invariant.

## tx.origin authentication

Never authorize with `tx.origin` (the original EOA). A malicious intermediary contract the user is tricked into calling will have the user as `tx.origin`, so a phishing contract can impersonate them. Use `msg.sender` for authorization.

## Bad randomness

On-chain values (`block.timestamp`, `blockhash`, `block.prevrandao`) are **predictable/influenceable by validators** and must not seed anything valuable (lotteries, NFT trait rolls, games). Attackers can compute or grind the outcome. Use a verifiable randomness oracle (e.g. Chainlink VRF) for any randomness that gates value.

## Front-running / MEV

The mempool is public and miners/searchers order transactions. An attacker can see your pending tx and place their own ahead (front-run) or around it (sandwich) for profit. Mitigations: commit-reveal schemes, slippage limits and deadlines on swaps, minimizing extractable value in the design, and private mempools/relays where appropriate. Assume anything profitable and observable will be MEV'd.

## Oracle and price manipulation

Don't read a price from a spot source an attacker can move within one transaction (e.g. a single DEX pool's instantaneous reserves) — flash-loan attacks exploit exactly this. Use robust oracles (Chainlink price feeds) or manipulation-resistant measures (TWAPs), and validate oracle freshness/staleness.

## Denial of service

- **Unbounded loops** over arrays that grow with user input can exceed the block gas limit and permanently brick a function — prefer mappings and pull-payments.
- **Push payments** that revert can block a whole batch — use pull-payments (see `contract-patterns.md`).
- A required external call to an address an attacker controls can be made to always revert — don't make liveness depend on an untrusted callee.

## Pre-deploy audit checklist

Walk this before any mainnet deployment:

- [ ] Every privileged function has an explicit, correct access-control check.
- [ ] `initialize()` (if upgradeable) cannot be called by an attacker; implementation initializers disabled.
- [ ] All functions with external calls follow Checks-Effects-Interactions and/or use `nonReentrant`.
- [ ] ETH is sent via `call` with the result checked; ERC-20s handled via `SafeERC20`.
- [ ] No `tx.origin` authorization; no on-chain randomness gating value; no spot-price oracle without manipulation resistance.
- [ ] No unbounded loops over user-growable arrays; payouts use the pull pattern.
- [ ] `unchecked` blocks are individually justified; downcasts use `SafeCast`.
- [ ] Inputs validated (zero-address checks, bounds); events emitted on state changes.
- [ ] `delegatecall`/proxy storage layout is correct and upgrade authority is restricted.
- [ ] Comprehensive tests incl. fuzz/invariant and explicit attacker scenarios; static analysis (Slither) run; for meaningful value, an independent professional audit.

Security is a tradeoff-laden discipline, not a checklist alone — but a contract that fails any item above is not ready. When reviewing, name the specific attack and the specific line/state-ordering that enables it.
