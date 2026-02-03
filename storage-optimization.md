# Storage Optimization

> **Sources**:
> - SSTORE patterns, RedBlackTreeLib, SSTORE2: [0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills)
> - Transient storage (EIP-1153): added based on Perplexity validation against MegaEVM spec
> - SSTORE cost formula corrected: 22,100 compute + 20,000×(m-1) storage (validated via MegaEVM spec)

## Why Storage Matters More on MegaETH

MegaETH's multidimensional gas model charges **storage gas** on top of compute gas for persistent writes. Zero-to-nonzero SSTORE costs 22,100 compute + 20,000×(m-1) storage gas, where m is a bucket multiplier. This makes naive storage patterns significantly more expensive than on standard Ethereum.

## Strategy Priority

1. **Avoid storage entirely** — use transient storage (TSTORE/TLOAD) for temporary data
2. **Reuse existing slots** — RedBlackTreeLib, fixed-size arrays
3. **Store as bytecode** — SSTORE2 for write-once data (free reads)
4. **Minimize new slots** — pack structs, use mappings carefully

## Transient Storage (EIP-1153)

Zero storage gas cost. Data cleared after transaction.

```solidity
// Reentrancy guard using transient storage (no storage gas!)
bytes32 constant LOCK_SLOT = keccak256("reentrancy.lock");

modifier nonReentrant() {
    assembly {
        if tload(LOCK_SLOT) { revert(0, 0) }
        tstore(LOCK_SLOT, 1)
    }
    _;
    assembly {
        tstore(LOCK_SLOT, 0)
    }
}
```

**When to use:** reentrancy guards, callback flags, flash loan state, any intra-transaction temporary data.

## Solady RedBlackTreeLib

Manages contiguous storage slots — insert/remove reuses slots instead of allocating new ones.

```solidity
import {RedBlackTreeLib} from "solady/src/utils/RedBlackTreeLib.sol";

contract OptimizedStorage {
    using RedBlackTreeLib for RedBlackTreeLib.Tree;
    RedBlackTreeLib.Tree private _tree;

    function insert(uint256 key) external {
        _tree.insert(key);
        // Reuses freed slots from previous removals
    }

    function remove(uint256 key) external {
        _tree.remove(key);
        // Slot available for reuse (no new storage gas on next insert)
    }
}
```

**When to use:** any collection that grows/shrinks (order books, queues, user lists).

## Avoid Dynamic Mappings for Growing Data

```solidity
// ❌ Expensive: each new key = new storage slot
mapping(address => uint256) public balances;

// ✅ Better: fixed-size or RedBlackTreeLib
uint256[100] public fixedBalances;
```

## SSTORE2: On-Chain Data Storage

Store large immutable data as contract bytecode. Writes cost ~10K gas/byte; reads are **free** (EXTCODECOPY).

```solidity
import {SSTORE2} from "solady/src/utils/SSTORE2.sol";

// Write (deploy data as bytecode)
address pointer = SSTORE2.write(data);

// Read (free in view calls)
bytes memory data = SSTORE2.read(pointer);
```

**When to use:** on-chain websites, NFT metadata, immutable configs, large write-once data.

### Chunking for Large Data

For data > 24KB, chunk into multiple contracts:

```javascript
const CHUNK_SIZE = 15000; // 15KB per chunk
function chunkData(data) {
    const chunks = [];
    for (let i = 0; i < data.length; i += CHUNK_SIZE) {
        chunks.push(data.slice(i, i + CHUNK_SIZE));
    }
    return chunks;
}
```

## Struct Packing

Pack variables into fewer storage slots:

```solidity
// ❌ 3 storage slots (3 × storage gas)
struct UserBad {
    uint256 balance;    // slot 0
    uint256 lastUpdate; // slot 1
    bool active;        // slot 2
}

// ✅ 2 storage slots
struct UserGood {
    uint256 balance;     // slot 0
    uint128 lastUpdate;  // slot 1 (packed)
    bool active;         // slot 1 (packed)
}
```
