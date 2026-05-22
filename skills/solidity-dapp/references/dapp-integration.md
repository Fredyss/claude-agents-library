# dApp Frontend Integration

Connecting a deployed contract to a web frontend — the "app" half of a dApp. The contract is the backend; the frontend reads its state, sends it transactions, and reacts to its events, all through the user's wallet. The recurring principle: **the user's wallet holds the keys and signs everything — your frontend never touches a private key.**

## Contents
- The pieces
- Library choice: ethers vs viem vs wagmi
- The ABI
- Providers and signers
- Connecting a wallet
- Reading vs writing
- Handling transactions and confirmations
- Listening to events and indexing
- Frontend-side pitfalls

## The pieces

A typical dApp frontend needs: a **wallet** (MetaMask, WalletConnect, Coinbase, etc.) that injects an EIP-1193 provider and signs transactions; the contract's **address** and **ABI**; an **RPC endpoint** (the wallet's, or a provider like Alchemy/Infura) to read chain state; and a JS library to tie them together.

## Library choice: ethers vs viem vs wagmi

- **ethers.js** — the long-standing, widely documented library. `Contract`, `Provider`, `Signer` abstractions. Safe default, especially for tutorials and existing codebases.
- **viem** — modern, lightweight, TypeScript-first with excellent type inference from the ABI. Increasingly the default for new projects.
- **wagmi** — React **hooks** built on viem (`useAccount`, `useReadContract`, `useWriteContract`, `useWaitForTransactionReceipt`) plus a connector ecosystem (often paired with RainbowKit/ConnectKit for the connect-wallet UI). Use it for React apps — it handles caching, reactivity, and connection state for you.

For a React dApp, **wagmi + viem** is the current mainstream stack; plain **ethers** is fine for non-React or simpler apps.

## The ABI

The **Application Binary Interface** is the JSON description of a contract's functions and events — the frontend needs it (plus the address) to encode calls and decode results. Get it from your compiler output (Foundry `out/`, Hardhat `artifacts/`) and import it into the frontend; keep it in sync with the deployed contract. Type-safe stacks (viem/wagmi) infer argument and return types directly from the ABI, catching mismatches at compile time.

## Providers and signers

- A **provider** is a read-only connection to the chain — use it for `view`/`pure` calls and reading state. No signing, no gas, no wallet popup.
- A **signer** (from the connected wallet) authorizes state-changing transactions. Any call that writes needs a signer; the wallet prompts the user to approve and pays gas.

Wire reads through a provider and writes through a signer.

## Connecting a wallet

Request access to the user's accounts (e.g. `eth_requestAccounts` via the injected provider, or wagmi's `connect`). Then:
- Surface the connected **address** and **chain** in the UI.
- **Check the network** — if the user is on the wrong chain, prompt them to switch (wallets expose a switch-chain request). Don't let users transact on the wrong network.
- React to account/chain **change events** so the UI updates when the user switches accounts or networks in their wallet.

## Reading vs writing

- **Reads** (`view`/`pure`) are free, instant, and need no wallet approval — call through the provider. Cache and refetch as needed (wagmi does this for you).
- **Writes** (state changes) are transactions: they cost gas, require wallet approval, and are **asynchronous** — you get a tx hash immediately but the change isn't real until the tx is mined. Design the UI around this latency.

## Handling transactions and confirmations

A robust write flow:
1. Submit the transaction; get a hash back. Show a "pending" state.
2. **Wait for the receipt** (`wait()` in ethers, `useWaitForTransactionReceipt` in wagmi) before treating the change as done.
3. On success, update the UI from fresh chain state or the emitted event; on failure (revert, user rejection, out-of-gas), show a clear, decoded error.

Always handle the **user-rejection** path (they declined in the wallet) — it's the most common "error" and isn't a bug. Consider estimating gas and simulating the call first to catch reverts before the user signs.

## Listening to events and indexing

Contracts communicate state changes via **events** (see `language.md`). The frontend can subscribe to live events for real-time updates and query historical logs to reconstruct state. For anything beyond simple cases, query a dedicated **indexer** (The Graph subgraph, or a hosted indexing service) rather than scanning logs in the browser — pulling large histories client-side is slow and unreliable. Indexers turn the event log into a fast, queryable API for your UI.

## Frontend-side pitfalls

- **Never put a private key or mnemonic in frontend code.** Signing happens in the user's wallet, always.
- **Don't trust the frontend for security.** Disabling a button is not access control — every check that matters must be enforced *in the contract*. The UI is a convenience, not a guard.
- **Handle decimals correctly.** Token amounts are integers scaled by `decimals` (usually 18). Convert at the UI edge (`parseUnits`/`formatUnits`); never do floating-point token math.
- **Account for confirmation latency and reorgs** — don't render a write as final before it's mined (and for high value, wait for several confirmations).
- **Validate the chain ID** so users don't accidentally transact on the wrong network.
- Surface clear, human-readable errors by decoding custom errors/revert reasons from the ABI rather than dumping raw RPC errors.
