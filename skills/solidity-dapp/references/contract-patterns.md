# Contract Patterns & Standards

Battle-tested patterns and the token standards every dApp leans on. The meta-rule: **for anything standardized, inherit an audited implementation (OpenZeppelin) rather than writing it yourself.** Re-implementing ERC-20 or access control by hand is a classic way to ship bugs that audited libraries already solved.

## Contents
- Token standards: ERC-20, ERC-721, ERC-1155
- OpenZeppelin building blocks
- Access control
- Pull-over-push payments
- Pausing / circuit breakers
- Factory pattern
- Upgradeable contracts (proxies)

## Token standards

- **ERC-20 — fungible tokens** (currencies, governance/utility tokens). Interface: `totalSupply`, `balanceOf`, `transfer`, `approve`, `allowance`, `transferFrom`, plus `Transfer`/`Approval` events. Watch the **approve/transferFrom** flow: a spender must be approved before pulling tokens. When *consuming* arbitrary ERC-20s, use OpenZeppelin's **`SafeERC20`** wrappers (`safeTransfer`, `safeTransferFrom`) — many real tokens don't return a bool or behave non-standardly, and naive calls silently fail.
- **ERC-721 — non-fungible tokens (NFTs)**. Each token has a unique `tokenId` with a single owner; `ownerOf`, `transferFrom`, `approve`, plus `tokenURI` (usually pointing to metadata JSON, often on IPFS). Use `_safeMint`/`safeTransferFrom` so tokens aren't stranded in contracts that can't handle them.
- **ERC-1155 — multi-token**. One contract manages many token types, fungible or not, with batch transfers. Efficient for games/marketplaces holding many item types.

Start every token from the OpenZeppelin base and add only your custom logic on top.

## OpenZeppelin building blocks

The de-facto standard library of audited contracts. Commonly composed:
- `ERC20`, `ERC721`, `ERC1155` — standard implementations, plus extensions (`ERC20Burnable`, `ERC20Permit`, `ERC721Enumerable`, `ERC721URIStorage`, …).
- `Ownable` / `Ownable2Step` — single-owner access control (`Ownable2Step` avoids fat-fingering ownership transfer to a wrong/dead address).
- `AccessControl` — role-based permissions for multi-role systems.
- `ReentrancyGuard` — the `nonReentrant` modifier.
- `Pausable` — emergency stop.
- `SafeERC20` — safe wrappers for interacting with external tokens.
- Proxy/upgrade utilities (`ERC1967Proxy`, `UUPSUpgradeable`, `TransparentUpgradeableProxy`) and `Initializable`.

Pin the OpenZeppelin version, and prefer their upgradeable variants (`@openzeppelin/contracts-upgradeable`) only when you actually use proxies.

## Access control

Almost every exploit ultimately traces to "someone called something they shouldn't have." Decide the model deliberately:
- **Single owner** — `Ownable`/`Ownable2Step` with an `onlyOwner` modifier on admin functions. Simplest; fine for many contracts.
- **Roles** — `AccessControl` with named roles (e.g. `MINTER_ROLE`, `PAUSER_ROLE`) granted/revoked by an admin role. Use when different actors have different powers.
- For high-value control, point ownership/admin at a **multisig** (e.g. Safe) or a timelock, not a single EOA.

Guard *every* privileged function. A missing modifier on one admin setter is a full compromise.

## Pull-over-push payments

Don't push ETH to many recipients in a loop (push payments): one recipient that reverts (or a gas spike) can brick the whole distribution, and looping over a growing list risks the block gas limit. Instead, **let recipients withdraw** (pull): record what each is owed, expose a `withdraw()` they call themselves. This isolates failures to the individual and is the standard pattern for refunds, dividends, and auction proceeds. OpenZeppelin's `PullPayment`/`Escrow` implement it.

## Pausing / circuit breakers

For contracts holding significant value, a `Pausable` "emergency stop" lets a trusted role halt sensitive operations if a bug or attack is detected — buying time to respond. Pair it with a clear, restricted authority over who can pause/unpause. Don't over-pause read paths users need for withdrawals.

## Factory pattern

A factory contract deploys other contracts (`new Child(...)`) and typically tracks them. Used for "one contract per user/market/pool" designs (e.g. a DEX deploying a pair contract per token pair). Keep per-instance state in the children; keep the registry in the factory. For many cheap clones, the **minimal proxy (EIP-1167 / OpenZeppelin `Clones`)** pattern deploys lightweight delegating copies of a single implementation.

## Upgradeable contracts (proxies)

Deployed code is immutable, so "upgrades" are achieved by splitting **a proxy (holds state and a pointer to logic) from an implementation (holds code)**; calls hit the proxy, which `delegatecall`s into the implementation, so state lives in the proxy while logic can be swapped. Common variants: **Transparent** proxy and **UUPS** (upgrade logic in the implementation; cheaper, now generally preferred).

Upgradeability adds serious footguns — handle with care:
- **No constructors** in implementations; use an `initialize()` function guarded by `Initializable` (and disable initializers in the implementation).
- **Never reorder or remove storage variables** between versions — the proxy's storage layout must stay compatible, or you'll corrupt state. Append-only, and use storage gaps or ERC-7201 namespaced layout. (0.8.35 adds an `erc7201` builtin to compute namespaced storage base slots.)
- Restrict who can upgrade (multisig/timelock), since the upgrade key can replace all logic.
- Watch for **storage collisions** and unintended `selfdestruct`/`delegatecall` exposure.

Upgradeability is powerful but trades away immutability's trust guarantees — only adopt it when the contract genuinely needs to evolve, and document the upgrade authority clearly.
