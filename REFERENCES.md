# References & Validation

This skill combines multiple sources. Every technical claim is traceable to a primary source.

## Primary Sources

### MegaEVM Specification (megaeth-labs/mega-evm)

The authoritative source for MegaETH's execution layer. All gas model and resource limit claims in this skill were validated against these docs:

| Document | URL | What it validates |
|----------|-----|-------------------|
| **Dual Gas Model** | [docs/DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) | Compute + storage gas separation, intrinsic gas (21K + 39K = 60K), SSTORE formula `20,000 × (m-1)`, LOG costs, calldata costs, Rex vs MiniRex differences |
| **Block & TX Limits** | [docs/BLOCK_AND_TX_LIMITS.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/BLOCK_AND_TX_LIMITS.md) | 7 resource limit types, Rex limits: 200M compute gas, 500K KV updates, 1K state growth, 12.5MB data |
| **Resource Accounting** | [docs/RESOURCE_ACCOUNTING.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/RESOURCE_ACCOUNTING.md) | How limits are tracked during execution |
| **MiniRex Spec** | [specs/MiniRex.md](https://github.com/megaeth-labs/mega-evm/blob/main/specs/MiniRex.md) | Volatile data access control (20M gas cap after block.timestamp), gas forwarding 98/100, contract size 512KB, SELFDESTRUCT disabled, oracle contract |
| **Rex Spec** | [specs/Rex.md](https://github.com/megaeth-labs/mega-evm/blob/main/specs/Rex.md) | Rex hardfork changes (production gas model) |
| **Intrinsic Gas Tests** | [tests/rex/intrinsic_gas.rs](https://github.com/megaeth-labs/mega-evm/blob/main/crates/mega-evm/tests/rex/intrinsic_gas.rs) | Test code confirming 60K intrinsic gas |
| **Storage Gas Tests** | [tests/rex/storage_gas.rs](https://github.com/megaeth-labs/mega-evm/blob/main/crates/mega-evm/tests/rex/storage_gas.rs) | Test code confirming SSTORE storage gas formula |
| **Gas Profiler Script** | [scripts/trace_opcode_gas.py](https://github.com/megaeth-labs/mega-evm/blob/main/scripts/trace_opcode_gas.py) | Opcode-level gas analysis tool |

### MegaETH Official Documentation

| Resource | URL | What it covers |
|----------|-----|----------------|
| **Docs home** | https://docs.megaeth.com | General MegaETH documentation |
| **Realtime API** | https://docs.megaeth.com/realtime-api | `eth_sendRawTransactionSync` / `realtime_sendRawTransaction` |
| **Testnet guide** | https://docs.megaeth.com/testnet | Carrot testnet setup |
| **Frontier (Mainnet)** | https://docs.megaeth.com/frontier | Mainnet information |

### Original MegaETH Developer Skill

| Resource | URL | What we used |
|----------|-----|--------------|
| **0xBreadguy/megaeth-ai-developer-skills** | https://github.com/0xBreadguy/megaeth-ai-developer-skills | Base chain-specific patterns: wallet operations, RPC methods, smart contract patterns, frontend patterns, security considerations, resource links. Testnet-verified by the original author. |

### Foundry Documentation

| Resource | URL | What we adapted |
|----------|-----|-----------------|
| **Foundry Prompting Guide** | https://getfoundry.sh/introduction/prompting/ | Testing methodology (unit/fuzz/invariant naming + handler pattern), project structure, naming conventions, deployment scripts, forge lint, dynamic test linking |
| **Forge Testing** | https://getfoundry.sh/forge/tests/overview | Test framework reference |
| **Forge Linting** | https://getfoundry.sh/forge/linting | `forge lint` for security/style |
| **Dynamic Test Linking** | https://getfoundry.sh/config/dynamic-test-linking | 10x+ faster compilation |
| **Fork Testing** | https://getfoundry.sh/forge/tests/fork-testing | Testing against live chains |
| **Deploying & Verifying** | https://getfoundry.sh/forge/deploying | Deployment scripts + verification |

### Ethereum Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| **EIP-7966** | https://ethereum-magicians.org/t/eip-7966-eth-sendrawtransactionsync-method/24640 | Synchronous transaction submission (instant receipts) |
| **EIP-1153** | https://eips.ethereum.org/EIPS/eip-1153 | Transient storage (TSTORE/TLOAD) — reduces storage gas |
| **EIP-6909** | https://eips.ethereum.org/EIPS/eip-6909 | Minimal multi-token standard |
| **EIP-2200** | https://eips.ethereum.org/EIPS/eip-2200 | SSTORE gas metering (referenced by MegaEVM dual gas model) |
| **EIP-7623** | https://eips.ethereum.org/EIPS/eip-7623 | Transaction floor cost (calldata pricing) |

### Libraries

| Library | URL | Usage |
|---------|-----|-------|
| **Solady** | https://github.com/Vectorized/solady | RedBlackTreeLib (storage optimization), SSTORE2 (bytecode storage), ERC6909 |
| **OpenZeppelin** | https://github.com/OpenZeppelin/openzeppelin-contracts | Access control, security patterns |
| **viem** | https://viem.sh | TypeScript Ethereum library |

### MegaETH Ecosystem

| Resource | URL | What it covers |
|----------|-----|----------------|
| **Token list** | https://github.com/megaeth-labs/mega-tokenlist | Verified token addresses, symbols, decimals |
| **Kyber Network aggregator** | https://docs.kyberswap.com/kyberswap-solutions/kyberswap-aggregator | DEX aggregator API for MegaETH swaps |
| **Envio HyperSync** | https://docs.envio.dev/docs/HyperSync/overview | High-performance historical data indexing |

## Validation Notes

### What changed from the original 0xBreadguy skill

| Claim in original skill | Correction | Source |
|--------------------------|------------|--------|
| SSTORE 0→nonzero "2M+ gas" | This was correct for MiniRex (`2,000,000 × multiplier`). Rex changed it to `20,000 × (multiplier - 1)` — zero storage gas when multiplier=1. | [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) |

### What was added (not in original skill)

| Addition | Source |
|----------|--------|
| Gas forwarding ratio 98/100 (vs Ethereum 63/64) | [MiniRex spec §2.4](https://github.com/megaeth-labs/mega-evm/blob/main/specs/MiniRex.md) |
| Per-tx resource limits (200M compute, 500K KV, 1K state growth, 12.5MB data) | [BLOCK_AND_TX_LIMITS.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/BLOCK_AND_TX_LIMITS.md) (Rex spec) |
| Transient storage (EIP-1153 TSTORE/TLOAD) for avoiding storage gas | [EIP-1153](https://eips.ethereum.org/EIPS/eip-1153) + recommended in [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) |
| Foundry config (foundry.toml, mega.etherscan.io verification) | [Foundry docs](https://getfoundry.sh/forge/deploying) adapted for MegaETH |
| Testing methodology (fuzz/invariant handlers) | [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/) adapted for MegaETH gas |
| Debugging guide (mega-evme + Foundry traces + common errors) | Combined from original skill + Foundry docs |
| MiniRex → Rex hardfork context | [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) historical reference section |
