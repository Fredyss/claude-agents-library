# Tooling, Testing & Gas

How contracts are built, tested, deployed, and optimized. The guiding principle: because there is no patching after deploy, **testing rigor is not optional** — it's the main thing standing between your code and a permanent exploit. Optimize gas only after correctness is proven.

## Contents
- Choosing a toolchain
- Foundry essentials
- Hardhat essentials
- Remix
- Testing strategy
- Static analysis
- Deployment and verification
- Gas optimization

## Choosing a toolchain

- **Foundry** (Rust-based: `forge`, `cast`, `anvil`) — tests are written *in Solidity*, runs are very fast, and it has first-class fuzzing and invariant testing built in. The modern default for contract-heavy projects and the one most current courses teach. Use it unless you have a reason not to.
- **Hardhat** (Node/JS/TS) — tests in JavaScript/TypeScript, huge plugin ecosystem, integrates naturally when the same repo holds a JS frontend. Strong choice for full-stack teams already in the Node world.
- **Remix** — browser IDE, zero setup; ideal for learning, quick prototypes, and inspecting/verifying a contract. Not where you run a serious test suite.

It's common to use Foundry for contracts/tests and a JS frontend separately; pick based on the team's center of gravity.

## Foundry essentials

- `forge init` scaffolds a project; dependencies via `forge install` (git submodules), e.g. OpenZeppelin.
- Tests live in `test/`, are contracts inheriting `forge-std/Test.sol`, with functions named `test_...` (and `testFuzz_...` for fuzzing, `invariant_...` for invariants). Cheatcodes via `vm.*`: `vm.prank(addr)` (set next `msg.sender`), `vm.expectRevert(...)`, `vm.warp`/`vm.roll` (time/block), `vm.deal` (set balance), `vm.expectEmit`.
- `forge test` runs everything; `-vvvv` shows traces; `forge test --gas-report` reports gas; `forge coverage` for coverage.
- `forge fmt` formats; `forge snapshot` tracks gas regressions; **fork testing** (`vm.createSelectFork`) runs tests against real mainnet state.
- `cast` for ad-hoc chain interaction; `anvil` is a local dev node.

## Hardhat essentials

- TS/JS config; tests with Mocha/Chai (often via `hardhat-toolbox`), using `ethers`/`viem` to deploy and call.
- Local node `npx hardhat node`; run tests `npx hardhat test`; scripts under `scripts/` for deploy.
- Rich plugins: gas reporter, contract verification, coverage, network management.

## Remix

Compile, deploy to a JS VM or injected wallet, and interact through a UI. Great for teaching the deploy/call loop and for poking at a deployed contract, plus flattening/verifying source.

## Testing strategy

Layer your tests; each catches a different class of bug:
1. **Unit tests** — every function's happy path and each revert condition (assert it reverts with the right error, with the right caller).
2. **Negative/authorization tests** — unauthorized callers are rejected; invalid inputs revert.
3. **Fuzz tests** — feed randomized inputs to surface edge cases (overflow boundaries, zero/empty, extreme values) you wouldn't enumerate by hand.
4. **Invariant tests** — assert properties that must hold across *any* sequence of calls (e.g. "sum of balances == totalSupply", "vault can always cover withdrawals"). The highest-value technique for stateful contracts.
5. **Attacker-scenario tests** — write a malicious contract that attempts reentrancy/abuse and assert it fails.
6. **Fork tests** — exercise integrations against real protocols/tokens on a forked network.

Aim for high coverage, but coverage is necessary not sufficient — invariants and adversarial tests catch what line coverage misses.

## Static analysis

Run automated analyzers as a cheap first pass: **Slither** (fast, catches many known bug classes and bad patterns) and optionally Mythril/symbolic tools. They produce false positives — triage, don't blindly obey — but they routinely catch missing checks, reentrancy, and shadowing. For anything holding meaningful value, an independent professional **audit** is the real bar; tools and tests don't replace it.

## Deployment and verification

- Deploy via a **script** (Foundry `forge script ... --broadcast`, or a Hardhat deploy script), never by hand for production — scripts are reproducible and reviewable.
- Manage secrets safely: keep private keys out of source; use a keystore/hardware wallet or environment-injected signer, and a multisig as the deployer/owner for high value.
- Deploy to a **testnet** (e.g. Sepolia) first and exercise it end-to-end.
- **Verify source** on the block explorer (Etherscan) so users can read and trust the code; verification also enables the explorer's read/write UI.
- Record deployed addresses and the exact compiler version/settings used.

## Gas optimization

Do this **last**, after correctness, and never trade away safety or clarity for it. High-leverage techniques, roughly in order:
- **Minimize storage writes** — `SSTORE` is among the most expensive ops. Cache `storage` reads in `memory` locals inside loops; write back once.
- **Pack storage** — group small types (`uint128`, `bool`, `address`) so they share 32-byte slots; order struct/state fields by size.
- **Use `calldata`** (not `memory`) for external reference-type params; mark functions `external` where possible.
- **Custom errors** instead of `require` strings; **`immutable`/`constant`** for values fixed at deploy/compile time (they're not stored in slots).
- **`unchecked`** for provably-safe arithmetic (e.g. loop counters) — with a justifying comment.
- Avoid unbounded loops; prefer events over storing data that only off-chain consumers need.
- Measure with `forge test --gas-report`/`forge snapshot` (or Hardhat gas reporter) — optimize against numbers, not hunches. (Solidity 0.8.35 ships an experimental SSA-based IR backend aimed at reducing stack-too-deep errors and compile time; the standard pipeline remains the default.)
