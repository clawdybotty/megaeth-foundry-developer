---
name: megaeth-foundry-developer
description: "Self-contained MegaETH + Foundry development skill. Covers the full lifecycle: wallet operations, smart contract development with MegaEVM-specific patterns, Foundry testing (unit/fuzz/invariant adapted for MegaETH gas model), deployment with verification, token swaps (Kyber Network), eth_sendRawTransactionSync (EIP-7966), JSON-RPC batching, WebSocket mini-block subscriptions, storage optimization (Solady RedBlackTreeLib, transient storage), multidimensional gas model, bridging from Ethereum, and debugging with mega-evme. Use when building on MegaETH, deploying contracts, writing tests, or managing wallets — no second skill needed."
---

# MegaETH + Foundry Development Skill

> **Sources & Attribution**
> - MegaETH chain-specific patterns: [0xBreadguy/megaeth-ai-developer-skills](https://github.com/0xBreadguy/megaeth-ai-developer-skills) (MIT)
> - Foundry patterns adapted from: [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/)
> - Gas model & resource limits validated via MegaEVM spec ([megaeth-labs/mega-evm](https://github.com/megaeth-labs/mega-evm)) and Perplexity-grounded research (Feb 2026)

## When to use this Skill

Use when the user asks about:
- Building anything on MegaETH (contracts, dApps, scripts)
- Foundry project setup targeting MegaETH
- Writing tests (unit, fuzz, invariant) for MegaETH contracts
- Deploying and verifying contracts on MegaETH
- Wallet operations, token swaps, bridging
- Gas optimization or storage patterns on MegaEVM
- RPC configuration, WebSocket subscriptions, real-time features
- Debugging failed transactions on MegaETH

## Chain Configuration

| Parameter | Mainnet | Testnet |
|-----------|---------|---------|
| Chain ID | 4326 | 6343 |
| HTTP RPC | `https://mainnet.megaeth.com/rpc` | `https://carrot.megaeth.com/rpc` |
| WebSocket | `wss://mainnet.megaeth.com/ws` | `wss://carrot.megaeth.com/ws` |
| Explorer | `https://mega.etherscan.io` | `https://megaeth-testnet-v2.blockscout.com` |
| Native Token | ETH | ETH |

## Critical MegaETH differences from standard EVM

Know these before writing any code:

1. **Intrinsic gas = 60,000** (not 21,000) — 21K compute + 39K storage gas
2. **Multidimensional gas** — compute gas + storage gas are separate dimensions
3. **SSTORE 0→nonzero** — 22,100 compute + 20,000×(m-1) storage gas (m = bucket multiplier). Expensive at high multipliers.
4. **Gas forwarding ratio: 98/100** (not 63/64 like Ethereum) — more gas available in nested calls
5. **Per-tx resource limits**: 200M compute gas, 500K KV updates, 1,000 state growth slots, 12.5MB data
6. **Base fee**: stable at 0.001 gwei, no EIP-1559 dynamics
7. **Volatile data**: after accessing `block.timestamp`/`block.number`/`blockhash()`, tx limited to 20M additional compute gas — access metadata late
8. **LOG opcodes**: quadratic cost above 4KB data
9. **Contract size**: 512KB max, calldata 128KB max
10. **Transient storage (EIP-1153)**: `TSTORE`/`TLOAD` available — use to avoid storage gas costs for temporary data

> Sources: Claims 1-2, 4-5 validated against MegaEVM spec. Claims 3, 6-10 from [0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills), testnet-verified pre-mainnet.

## Default stack decisions

1. **Transaction submission**: `eth_sendRawTransactionSync` (EIP-7966) — instant receipt in <10ms
2. **RPC batching**: Multicall (`aggregate3`) preferred over JSON-RPC batching (since v2.0.14)
3. **WebSocket**: keepalive every 30s via `eth_chainId`; 50 connections/endpoint, 10 subs/connection
4. **Storage**: Solady RedBlackTreeLib over mappings; prefer transient storage for temp data
5. **Gas**: hardcode when possible; always use remote `eth_estimateGas` (MegaEVM costs differ)
6. **Testing**: Foundry with `--skip-simulation` (Foundry ignores storage gas)
7. **Debugging**: mega-evme CLI for transaction replay and gas profiling

## Operating procedure

### 1. Classify the task layer
- **Foundry setup** → read [foundry-config.md](foundry-config.md)
- **Wallet/transfers/swaps/bridging** → read [wallet-operations.md](wallet-operations.md)
- **Frontend/WebSocket** → read [frontend-patterns.md](frontend-patterns.md)
- **RPC/transaction** → read [rpc-methods.md](rpc-methods.md)
- **Smart contracts** → read [smart-contracts.md](smart-contracts.md)
- **Storage optimization** → read [storage-optimization.md](storage-optimization.md)
- **Gas questions** → read [gas-model.md](gas-model.md)
- **Testing** → read [testing.md](testing.md)
- **Deployment** → read [foundry-config.md](foundry-config.md) (deployment section)
- **Security** → read [security.md](security.md)
- **Debugging** → read [debugging.md](debugging.md)
- **Reference links** → read [resources.md](resources.md)

### 2. Implement with MegaETH-specific correctness
Always be explicit about:
- Chain ID (4326 mainnet, 6343 testnet)
- Gas limit (hardcode when possible, use remote estimation otherwise)
- Base fee (0.001 gwei, no buffer)
- Storage costs (new slots are expensive — prefer transient storage or slot reuse)
- Volatile data limits (20M gas after `block.timestamp` access)
- Gas forwarding (98/100 — more gas available in nested calls than on Ethereum)

### 3. Deliverables expectations
When implementing changes, provide:
- Exact files changed + diffs
- Commands to build/test/deploy
- Gas cost notes for storage-heavy operations
- foundry.toml configuration if setting up a new project
