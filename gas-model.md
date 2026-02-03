# Gas Model

> **Sources**: All claims validated against [megaeth-labs/mega-evm](https://github.com/megaeth-labs/mega-evm):
> - Dual gas model: [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md)
> - Resource limits: [BLOCK_AND_TX_LIMITS.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/BLOCK_AND_TX_LIMITS.md)
> - Volatile data, gas forwarding: [MiniRex spec](https://github.com/megaeth-labs/mega-evm/blob/main/specs/MiniRex.md)
> - Base fee, LOG costs: [0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills), testnet-verified

## Multidimensional Gas Model

MegaEVM separates gas into two dimensions:

| Dimension | What it covers | Notes |
|-----------|---------------|-------|
| **Compute gas** | Execution (identical to standard EVM opcode costs) | Same as Ethereum |
| **Storage gas** | Persistent data operations (SSTORE, account creation, contract deployment, logs, calldata) | MegaETH-specific |

**Total transaction gas = compute gas + storage gas.** Most operations incur only compute gas. Storage-intensive operations add storage gas on top.

## Intrinsic Gas

| Component | Gas | Notes |
|-----------|-----|-------|
| Compute intrinsic | 21,000 | Same as Ethereum |
| Storage intrinsic | 39,000 | MegaETH-specific |
| **Total intrinsic** | **60,000** | Every transaction pays this minimum |

RPC rejects transactions with gas limit below 60,000 with "intrinsic gas too low".

## SSTORE Costs

> **Source**: [DUAL_GAS_MODEL.md](https://github.com/megaeth-labs/mega-evm/blob/main/docs/DUAL_GAS_MODEL.md) — Rex hardfork specification

| Operation | Compute Gas | Storage Gas | Notes |
|-----------|-------------|-------------|-------|
| Zero → nonzero | 22,100 (cold) | 20,000 × (m-1) | m = bucket multiplier (SALT bucket capacity / MIN_BUCKET_SIZE) |
| Nonzero → nonzero | 5,000 (cold) | 0 | Same as standard EVM |
| Nonzero → zero | 5,000 (cold) | 0 | No refund for set-and-reset in same tx (use transient storage instead) |

The bucket multiplier `m` depends on SALT bucket capacity ratio — when m=1 (standard bucket), storage gas is **zero**. At higher multipliers, cost scales linearly.

**Historical note:** The original MegaETH skill referenced "2M+ gas" for SSTORE — this was correct under the **MiniRex** hardfork (`2,000,000 × multiplier`). The **Rex** hardfork changed the formula to `20,000 × (multiplier - 1)`, which is significantly cheaper and zero at multiplier=1.

⚠️ **No refund for set-and-reset:** Unlike standard EVM, repeatedly setting and clearing a storage slot in one transaction accumulates storage gas. Use transient storage (TSTORE/TLOAD) for temporary data.

## Transient Storage (EIP-1153)

> **Source**: Identified as missing from original skill via Perplexity validation.

`TSTORE`/`TLOAD` are available on MegaETH. Use for temporary data to avoid storage gas entirely:

```solidity
// ❌ Expensive: persistent storage for temporary data
mapping(address => bool) private _entered;

// ✅ Free (no storage gas): transient storage
// Uses EIP-1153 TSTORE/TLOAD — cleared after transaction
assembly {
    tstore(slot, 1)  // Set
    let val := tload(slot)  // Read
}
```

Use cases: reentrancy guards, callback flags, flash loan state, any data needed only within a single transaction.

## Gas Forwarding

> **Source**: Validated against MegaEVM spec via Perplexity research.

| Chain | Gas forwarding ratio |
|-------|---------------------|
| Ethereum | 63/64 (~98.4%) |
| **MegaETH** | **98/100 (98%)** |

Slightly less gas is available in nested calls on MegaETH than Ethereum. This matters for deeply nested call chains.

## Per-Transaction Resource Limits

> **Source**: Validated against MegaEVM spec via Perplexity research.

| Resource | Tx Limit | Notes |
|----------|----------|-------|
| Compute gas | 200M | Much higher than Ethereum |
| KV updates | 500,000 | Storage writes per tx |
| State growth | 1,000 | New storage slots per tx |
| Data size | 12.5 MB | Total data per tx |

These limits are important for storage-heavy contracts — invariant tests with many storage writes may hit the 500K KV updates limit.

## Base Fee

| Parameter | Value | Notes |
|-----------|-------|-------|
| Base fee | 0.001 gwei (10⁶ wei) | Stable, no EIP-1559 adjustment |
| Priority fee | 0 | Ignored unless congested |

```javascript
// Correct gas price setting
const gasPrice = 1000000n; // 0.001 gwei in wei

// ❌ Wrong: viem adds 20% buffer
const gasPrice = await publicClient.getGasPrice(); // Returns 1.2M wei
```

## Gas Estimation

### When to hardcode
For known operations, hardcode MegaETH-specific gas limits:

```javascript
const gasLimits = {
    transfer: 60000n,       // NOT 21000
    erc20Transfer: 100000n,
    erc20Approve: 80000n,
    swap: 350000n,
};
```

### When to use remote estimation
For any non-trivial or unknown operation, use `eth_estimateGas`:

```javascript
// ✅ Remote estimation (accounts for storage gas)
const gas = await client.request({
    method: 'eth_estimateGas',
    params: [{ from, to, data }]
});

// ❌ Local simulation (ignores storage gas)
// Foundry/Hardhat local estimates are WRONG for MegaETH
```

For Foundry: always use `--skip-simulation` with explicit `--gas-limit`.

## Volatile Data Access Limit

After accessing block metadata (`block.timestamp`, `block.number`, `blockhash(n)`), the transaction is limited to **20M additional compute gas**.

```solidity
// ❌ Problematic: heavy work after timestamp access
function process() external {
    uint256 ts = block.timestamp;  // Triggers 20M limit
    for (uint i = 0; i < 10000; i++) { /* heavy work */ }
}

// ✅ Better: access metadata late
function process() external {
    for (uint i = 0; i < 10000; i++) { /* heavy work first */ }
    emit Processed(block.timestamp);  // Access at the end
}
```

For high-precision timestamps without the gas penalty, use the oracle:

```solidity
ITimestampOracle constant ORACLE =
    ITimestampOracle(0x6342000000000000000000000000000000000002);
uint256 ts = ORACLE.timestamp(); // Microseconds, separate accounting
```

## LOG Opcode Costs

After a DoS fix, LOG opcodes have **quadratic cost above 4KB data**:

```solidity
// ❌ Expensive: large event data
event LargeData(bytes data); // If data > 4KB, quadratic cost

// ✅ Better: emit hash, store off-chain
event DataStored(bytes32 indexed hash);
```
