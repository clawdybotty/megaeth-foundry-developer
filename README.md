# MegaETH + Foundry Developer Skill

A unified development skill for building on MegaETH with Foundry — from contract design through testing, deployment, and debugging.

## Why a combined skill?

### The problem with two separate skills

MegaETH is not a standard EVM chain. Its [multidimensional gas model](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md), resource limits, and execution semantics mean that generic Foundry knowledge produces subtly wrong output, and MegaETH chain knowledge without a Foundry testing methodology produces untested code.

If an AI agent loads **only the MegaETH skill** (the [original 0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills)):
- It knows *what* makes MegaETH different (storage gas, volatile data limits, 60K intrinsic gas)
- It can send transactions, set up wallets, build frontends
- But it has no structured approach to **testing** contracts against MegaETH's gas model
- It doesn't know Foundry conventions (naming, fuzz/invariant patterns, handler architecture)
- Deployment scripts won't include `--skip-simulation`, leading to wrong gas estimates

If an agent loads **only a generic Foundry skill** (e.g. from [getfoundry.sh/introduction/prompting](https://getfoundry.sh/introduction/prompting/)):
- It writes well-structured tests with proper naming and fuzz/invariant patterns
- It knows `forge script`, `forge verify-contract`, `forge lint`
- But local simulation uses **standard EVM gas**, which is wrong for MegaETH (different intrinsic gas, separate storage gas dimension)
- It won't know to use `--skip-simulation` or remote `eth_estimateGas`
- Storage optimization patterns (RedBlackTreeLib, transient storage) won't factor into contract design
- It has no awareness of volatile data limits, 98/100 gas forwarding, or per-tx resource caps

**Two separate skills** would require the agent to load both and mentally merge them — hoping it correctly reconciles conflicts (like Foundry's gas simulation being wrong for MegaETH). The merge happens in the agent's context window, not in validated documentation.

### What the combined skill does

This skill pre-integrates the merge. Every Foundry pattern is already adapted for MegaETH's execution model:

| Area | What's integrated |
|------|-------------------|
| **Gas model** | Foundry's gas reports annotated with MegaETH's compute + storage split. Agent knows `forge test` gas numbers are approximate. |
| **Testing** | Fuzz and invariant patterns from Foundry's prompting guide, with MegaETH-specific assertions (storage gas bounds, volatile data budget checks). |
| **Deployment** | `forge script` commands always include `--skip-simulation` and `--gas-limit`. Verification targets `mega.etherscan.io`. |
| **Debugging** | Foundry's `-vvvv` traces + MegaETH's `mega-evme` CLI as complementary tools, with a mapping of when to use which. |
| **Storage** | Contract design patterns (RedBlackTreeLib, transient storage, SSTORE2) wired into the testing methodology — the agent tests for storage costs, not just logic correctness. |
| **foundry.toml** | Pre-configured with MegaETH RPC endpoints, etherscan verification, and Foundry's `dynamic_test_linking` for fast compilation. |

The result: an agent using this skill writes MegaETH-optimized contracts, tests them with Foundry best practices, deploys with correct gas settings, and debugs with the right tools — without needing to resolve conflicts between two knowledge sources.

## What comes from where

This skill combines three sources, each contributing distinct expertise:

### From [0xBreadguy/megaeth-ai-developer-skills](https://github.com/0xBreadguy/megaeth-ai-developer-skills) (MIT)

The original MegaETH developer skill, testnet-verified by the author. Provides all chain-specific knowledge:

- Wallet operations (setup, transfers, swaps via Kyber, bridging)
- RPC methods reference (`eth_sendRawTransactionSync`, batching, WebSocket patterns)
- Smart contract patterns (predeploys, volatile data, oracle contract)
- Frontend patterns (React/Next.js, WebSocket subscriptions, real-time UX)
- Storage optimization (RedBlackTreeLib, SSTORE2, slot reuse)
- Security considerations (MegaETH-specific audit checklist)
- Resource links (explorer, token list, docs, ecosystem)

### From [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/)

The official Foundry guidance for AI agents. Provides development methodology:

- Project structure and naming conventions
- Test naming patterns (`test_`, `testFuzz_`, `invariant_`, `testFork_`)
- Invariant testing with handler contracts
- Deployment scripts (`forge script`)
- Contract verification (`forge verify-contract`)
- Linting (`forge lint`)
- Dynamic test linking for fast builds

### From [MegaEVM spec](https://github.com/megaeth-labs/mega-evm) (validated Feb 2026)

Primary source for gas model correctness. Used to validate and correct claims:

- **Intrinsic gas = 60,000** (21K compute + 39K storage) — confirmed via [intrinsic_gas.rs tests](https://github.com/megaeth-labs/mega-evm/blob/main/crates/mega-evm/tests/rex/intrinsic_gas.rs)
- **SSTORE formula**: 22,100 compute + 20,000×(m-1) storage gas — corrected from original "2M+" (which was accurate for MiniRex, changed in Rex hardfork)
- **Per-tx resource limits**: 200M compute, 500K KV updates, 1K state growth, 12.5MB data — from [BLOCK_AND_TX_LIMITS.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/BLOCK_AND_TX_LIMITS.md)
- **Gas forwarding ratio 98/100** — from [MiniRex spec](https://github.com/megaeth-labs/mega-evm/blob/main/specs/MiniRex.md)
- **Transient storage (EIP-1153)** recommended for avoiding storage gas — from [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md)

Full source traceability: see [REFERENCES.md](REFERENCES.md).

## Files added vs original skill

| File | Status | Purpose |
|------|--------|---------|
| `foundry-config.md` | **New** | `foundry.toml` template, project structure, naming conventions, deploy scripts, verification, linting |
| `debugging.md` | **New** | `mega-evme` CLI + Foundry debugging commands + common error table |
| `testing.md` | **Rewritten** | Was a brief debugging guide → now full Foundry testing methodology (unit/fuzz/invariant/fork) adapted for MegaETH |
| `gas-model.md` | **Expanded** | Added multidimensional gas explainer, corrected SSTORE formula, added source citations |
| `storage-optimization.md` | **Expanded** | Added transient storage (EIP-1153), corrected cost formula, added strategy priority |
| `REFERENCES.md` | **New** | Full source traceability for every claim |
| `SKILL.md` | **Rewritten** | Unified operating procedure routing to all files, MegaETH differences upfront |
| Other files | **Source-attributed** | Added source attribution headers; content from 0xBreadguy skill preserved |

## When to use this vs the original

| Situation | Recommendation |
|-----------|---------------|
| Building contracts + testing with Foundry | **Use this skill** — Foundry patterns pre-integrated |
| Quick wallet/transfer/swap operations only | Either works — this skill includes all wallet operations |
| Frontend-only work (no contracts) | Either works — same frontend patterns |
| Using Hardhat or Remix instead of Foundry | Use the **original skill** — Foundry patterns aren't relevant |
| Need the leanest possible context | Use the **original skill** — fewer files, smaller footprint |

## Installation

```bash
# Copy to your agent's skills directory
cp -r megaeth-foundry-developer/ /path/to/your/skills/
```

## License

MIT — same as both source skills.
