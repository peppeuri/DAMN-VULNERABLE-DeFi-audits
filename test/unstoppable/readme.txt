# Damn Vulnerable DeFi v4 - Unstoppable (Solved)

## Challenge
- Project: Damn Vulnerable DeFi v4
- Challenge: Unstoppable
- Goal: Halt the tokenized vault so it stops offering flash loans, starting with 10 DVT.

## Context
- Vault: `UnstoppableVault` (ERC4626-based) holding 1,000,000 DVT and offering free flash loans during the grace period.
- Monitor: `UnstoppableMonitor` performs on-chain liveness checks via `checkFlashLoan()`; on failure it pauses the vault and transfers ownership to the deployer.
- Player initial balance: 10 DVT.

## Exploit Summary
The vault enforces a strict accounting invariant in `flashLoan()`:
`convertToShares(totalSupply) == totalAssets()` (where `totalAssets()` is `asset.balanceOf(address(this))`).

By sending the player’s full 10 DVT directly to the vault with a plain ERC20 `transfer` (bypassing `deposit()`), `totalAssets()` increases while `totalSupply` remains unchanged. Due to Solmate’s integer-division share math, `convertToShares(totalSupply)` no longer equals the inflated balance, so `flashLoan()` reverts with `InvalidBalance()`. The monitor catches the revert, emits `FlashLoanStatus(false)`, pauses the vault, and takes ownership—causing a permanent DoS of the flash-loan facility.

## PoC (test/unstoppable/Unstoppable.t.sol)
```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);
}
```

## Result
- Flash loan checks fail permanently.
- Vault is paused and ownership transferred to the deployer (validated by `_isSolved()`).

## Artifacts
- Exploit test: `test/unstoppable/Unstoppable.t.sol`
- Audit write-up: `test/unstoppable/Unstoppable-Audit.md`
