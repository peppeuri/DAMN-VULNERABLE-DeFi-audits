# Unstoppable Vault - Security Audit Write-up

## Vulnerability Type
**Denial of Service (DoS) / ERC4626 Accounting Manipulation**

## Attack Vector
1. The attacker starts with `10e18` DVT (player balance) while the vault holds `1_000_000e18` DVT and has minted the same amount of shares.
2. Instead of calling `deposit()`, the attacker executes a plain ERC20 `transfer` directly to the vault contract: `token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE)`.
3. This increases the vault’s token balance (`totalAssets() == asset.balanceOf(address(this))`) without minting any new shares, so `totalSupply` remains unchanged.
4. On the next monitored flash loan, `UnstoppableMonitor.checkFlashLoan()` calls `vault.flashLoan(...)`.
5. `UnstoppableVault.flashLoan()` enforces a strict accounting invariant and reverts with `InvalidBalance()`.
6. The monitor catches the failure, emits `FlashLoanStatus(false)`, pauses the vault, and transfers ownership to the deployer—permanently halting the flash-loan facility.

Relevant exploit (from `test/unstoppable/Unstoppable.t.sol`):
```solidity
token.transfer(address(vault), INITIAL_PLAYER_TOKEN_BALANCE);
```

## Root Cause Analysis
`UnstoppableVault` mixes two accounting sources:
- **Actual asset balance:** `totalAssets()` returns `asset.balanceOf(address(this))` (`UnstoppableVault.sol:71-73`).
- **Share-based accounting:** `totalSupply` is updated only through ERC4626 deposit/mint/withdraw/redeem flows.

`flashLoan()` performs a strict equality check before serving loans (`UnstoppableVault.sol:84-85`):
```solidity
uint256 balanceBefore = totalAssets();
if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();
```

Using Solmate’s ERC4626 math (`FixedPointMathLib.mulDivDown`):
- `convertToShares(assets) = assets * totalSupply / totalAssets()`
- Therefore, `convertToShares(totalSupply) = totalSupply * totalSupply / totalAssets()`

In a consistent state, deposits made `totalAssets() == totalSupply` (1:1 initially), so:
- `convertToShares(totalSupply) = totalSupply * totalSupply / totalSupply = totalSupply == totalAssets()` → invariant holds.

After a direct transfer of `Δ = 10e18` DVT:
- `totalAssets() = balanceBefore = 1_000_000e18 + 10e18`
- `totalSupply = 1_000_000e18` (unchanged)

The invariant becomes:
- `convertToShares(totalSupply) = (1_000_000e18 * 1_000_000e18) / (1_000_000e18 + 10e18)`

Due to integer division, this value is truncated down and is strictly less than `balanceBefore` (and no longer equal). The strict equality comparison therefore fails even though there is no insolvency and sufficient liquidity exists. Because anyone can force-send ERC20 tokens to the vault address, this broken invariant is externally triggerable by an unprivileged attacker.

## Impact
- **Permanent Denial of Service** of the vault’s flash-loan functionality: every `flashLoan()` call reverts at the invariant check.
- **Operational shutdown:** the monitoring contract interprets the failure as a critical issue, pauses the vault, and transfers ownership to the deployer (`UnstoppableMonitor.sol:41-52`), freezing normal operation until manual intervention/redesign.
- **No fund theft required:** the attack is low-cost (donating existing tokens) and does not rely on privileged access or complex prerequisites.

## Remediation
1. **Remove the strict equality invariant in `flashLoan()` (recommended).** Flash loans should be constrained by available liquidity and ERC4626/ERC3156 semantics, not by assuming `balanceOf(vault)` equals share-converted supply. A direct donation must not brick protocol functionality.
2. **Do not use `asset.balanceOf(address(this))` as an integrity invariant against shares** when unsolicited transfers are possible. If internal consistency checks are required, track assets via internal accounting updated only in trusted entry points (`afterDeposit`, `beforeWithdraw`) rather than the raw token balance.
3. **Prefer robust comparisons if a sanity check is still desired.** For example, validate against internally tracked assets or use non-strict/tolerance-aware logic; never rely on exact equality that third parties can break by donating tokens.
4. **Harden monitoring behavior.** Treat a single flash-loan revert as an alert/metric, not an automatic pause-and-ownership-transfer trigger, unless the condition is provably indicative of compromise and cannot be griefed.

**Illustrative fix (conceptual):**
```solidity
// Remove this brittle check:
// uint256 balanceBefore = totalAssets();
// if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance();

// Instead, rely on liquidity/flow checks already inherent to transfer + repayment.
uint256 balanceBefore = totalAssets();
require(amount <= balanceBefore, "INSUFFICIENT_LIQUIDITY");
```
This prevents donation-based griefing from causing a protocol-wide flash-loan DoS while preserving solvency guarantees.
