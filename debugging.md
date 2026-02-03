# Debugging MegaETH Transactions

> **Source**: [0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills) + Foundry debugging commands from [Foundry docs](https://getfoundry.sh/).

## mega-evme CLI

Transaction replay and gas profiling tool from MegaETH team:

```bash
git clone https://github.com/megaeth-labs/mega-evm
cd mega-evm/bin/mega-evme
cargo build --release

# Replay a transaction
./mega-evme replay --tx-hash <hash> --rpc https://mainnet.megaeth.com/rpc

# Profile gas by opcode
./mega-evme trace --tx-hash <hash> --rpc https://mainnet.megaeth.com/rpc
```

Gas profiler script (opcode-level analysis):
https://github.com/megaeth-labs/mega-evm/blob/main/scripts/trace_opcode_gas.py

## Foundry Debugging

```bash
# Detailed test traces
forge test -vvvv --skip-simulation

# Gas estimation via cast
cast estimate \
    --from 0xYourAddress \
    --value 0.001ether \
    0xRecipient \
    --rpc-url https://mainnet.megaeth.com/rpc

# Read contract state at block
cast call --block 12345 0xContract "balanceOf(address)(uint256)" 0xAddress \
    --rpc-url https://mainnet.megaeth.com/rpc

# Get gas price (always 0.001 gwei on MegaETH)
cast gas-price --rpc-url https://mainnet.megaeth.com/rpc
```

## Common Errors

| Error | Cause | Solution |
|-------|-------|---------|
| "intrinsic gas too low" | Gas limit < 60,000 | Set gas limit ≥ 60,000 (MegaETH intrinsic) |
| "out of gas" after block.timestamp | Volatile data access limit | Restructure: access block metadata late in execution |
| Wrong gas in Foundry simulation | Foundry uses standard EVM gas | Use `--skip-simulation` + remote estimation |
| "nonce too low" | Transaction already executed | Check receipt — don't retry |
| "already known" | Transaction still pending | Wait for confirmation |
| "insufficient funds" | Not enough ETH | Check balance, fund wallet |
| Large event unexpectedly expensive | LOG quadratic cost > 4KB | Emit hash instead of raw data |
| Storage write unexpectedly expensive | Zero-to-nonzero SSTORE with high bucket multiplier | Use transient storage, RedBlackTreeLib, or SSTORE2 |
| Deep call chain runs out of gas | Gas forwarding ratio 98/100 | Budget gas accounting for 2% loss per call depth |
