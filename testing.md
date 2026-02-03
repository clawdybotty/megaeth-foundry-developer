# Testing on MegaETH with Foundry

> **Sources**:
> - Testing methodology (naming, fuzz, invariant handlers): [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/)
> - MegaETH-specific adaptations (gas model, fork testing, skip-simulation): [0xBreadguy skill](https://github.com/0xBreadguy/megaeth-ai-developer-skills) + MegaEVM spec validation

## Key Rule: Skip Simulation

MegaEVM has different gas costs than standard EVM. Foundry's local simulation uses standard costs, giving **wrong gas estimates**. Always:

```bash
# ⚠️ Use --skip-simulation for any on-chain interaction
forge test --fork-url https://carrot.megaeth.com/rpc --skip-simulation
forge script Deploy.s.sol --skip-simulation --gas-limit 5000000
```

## Unit Tests

```solidity
contract MyContractTest is Test {
    MyContract target;

    function setUp() public {
        // Fork MegaETH testnet for realistic gas costs
        vm.createSelectFork("https://carrot.megaeth.com/rpc");
        target = new MyContract();
    }

    // ✅ Positive case
    function test_Deposit_UpdatesBalance() public {
        target.deposit{value: 1 ether}();
        assertEq(target.balances(address(this)), 1 ether, "Balance not updated");
    }

    // ✅ Revert case
    function test_RevertWhen_DepositZero() public {
        vm.expectRevert("Amount must be > 0");
        target.deposit{value: 0}();
    }

    // ✅ Event emission
    function test_Deposit_EmitsEvent() public {
        vm.expectEmit(true, false, false, true);
        emit Deposited(address(this), 1 ether);
        target.deposit{value: 1 ether}();
    }

    // ✅ Gas measurement (MegaETH-specific)
    function test_Deposit_GasCost() public {
        uint256 gasBefore = gasleft();
        target.deposit{value: 1 ether}();
        uint256 gasUsed = gasBefore - gasleft();
        console.log("Deposit gas (MegaETH):", gasUsed);
        // Note: includes both compute + storage gas dimensions
    }
}
```

**Rules:**
- Never place assertions in `setUp()`
- Use descriptive assertion messages: `assertEq(result, expected, "error message")`
- Test state changes, event emissions, and return values
- Fork MegaETH for realistic gas costs

## Fuzz Tests

```solidity
function testFuzz_Deposit(uint96 amount) public {
    // ⚠️ Use uint96 to avoid overflow (not uint256)
    vm.assume(amount > 0.001 ether);
    vm.assume(amount < 1000 ether);

    deal(address(this), amount);
    target.deposit{value: amount}();

    assertEq(target.balances(address(this)), amount, "Balance mismatch");
}

// Use fixtures for MegaETH-specific edge cases
uint256[] public gasFixtures = [60000, 100000, 200000000]; // intrinsic, typical, tx limit
```

**MegaETH fuzz considerations:**
- Bound amounts to realistic ranges (MegaETH gas is cheap — large values are valid)
- Use `vm.assume()` to exclude invalid inputs, not early returns
- Test around MegaETH-specific boundaries: 60K intrinsic gas, 500K KV updates, 20M volatile data limit
- Configure runs: `forge test --fuzz-runs 1000`

## Invariant Tests with Handlers

> Source: Handler pattern from [Foundry Prompting Guide](https://getfoundry.sh/introduction/prompting/), adapted for MegaETH storage gas considerations.

```solidity
// Handler: bounded actions with ghost variable tracking
contract VaultHandler {
    Vault public vault;
    IERC20 public asset;

    // Ghost variables — track state across calls
    uint256 public ghost_depositSum;
    uint256 public ghost_withdrawSum;

    // Actor management
    address[] public actors;
    address internal currentActor;

    modifier useActor(uint256 actorSeed) {
        currentActor = actors[bound(actorSeed, 0, actors.length - 1)];
        vm.startPrank(currentActor);
        _;
        vm.stopPrank();
    }

    constructor(Vault _vault, IERC20 _asset) {
        vault = _vault;
        asset = _asset;
        for (uint i = 0; i < 5; i++) {
            actors.push(makeAddr(string(abi.encode("actor", i))));
        }
    }

    function deposit(uint256 assets, uint256 actorSeed) external useActor(actorSeed) {
        assets = bound(assets, 0, 1e30);
        deal(address(asset), currentActor, assets);
        asset.approve(address(vault), assets);
        vault.deposit(assets, currentActor);
        ghost_depositSum += assets;
    }

    function withdraw(uint256 shares, uint256 actorSeed) external useActor(actorSeed) {
        shares = bound(shares, 0, vault.balanceOf(currentActor));
        if (shares == 0) return;
        uint256 assets = vault.redeem(shares, currentActor, currentActor);
        ghost_withdrawSum += assets;
    }
}

// Invariant test contract
contract VaultInvariantTest is Test {
    Vault vault;
    MockERC20 asset;
    VaultHandler handler;

    function setUp() external {
        // Fork MegaETH for accurate gas costs
        vm.createSelectFork("https://carrot.megaeth.com/rpc");

        asset = new MockERC20();
        vault = new Vault(asset);
        handler = new VaultHandler(vault, asset);
        targetContract(address(handler));
    }

    function invariant_depositsGeWithdrawals() external view {
        assertGe(handler.ghost_depositSum(), handler.ghost_withdrawSum());
    }

    function invariant_totalAssetsBacksSupply() external view {
        assertGe(vault.totalAssets(), vault.totalSupply());
    }
}
```

**MegaETH invariant considerations:**
- Fork MegaETH in setUp for accurate storage gas behavior
- Storage-heavy invariants may hit the 500K KV updates limit — bound handler actions accordingly
- Be aware of the 98/100 gas forwarding ratio (more gas available in nested calls than Ethereum's 63/64)

## Fork Tests

```solidity
function testFork_SwapOnMegaETH() public {
    // Fork mainnet
    vm.createSelectFork("https://mainnet.megaeth.com/rpc");

    // Interact with deployed contracts
    IERC20 weth = IERC20(0x4200000000000000000000000000000000000006);
    // ... test against live state
}
```

## Running Tests

```bash
# All tests
forge test --skip-simulation

# Specific test
forge test --match-test test_Deposit --skip-simulation

# With traces (debug failing tests)
forge test -vvvv --skip-simulation

# Fuzz with custom runs
forge test --fuzz-runs 10000 --skip-simulation

# Coverage report
forge coverage --skip-simulation

# Gas snapshot
forge snapshot --skip-simulation
```

## Common Test Failures on MegaETH

| Error | Cause | Solution |
|-------|-------|---------|
| "intrinsic gas too low" | Gas limit < 60,000 | Set `--gas-limit 5000000` or use remote estimation |
| Wrong gas in local sim | Foundry uses standard EVM costs | Use `--skip-simulation` + fork testing |
| "out of gas" after timestamp | Volatile data access limit (20M) | Restructure: access `block.timestamp` late |
| Storage test unexpected cost | Multidimensional gas not simulated locally | Fork MegaETH for accurate storage gas |
