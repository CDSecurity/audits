# Aave Lending Protocol Security Checklist (V2 + V3)

## Purpose & Usage

This checklist merges insights from **Aave V2** and **Aave V3** to cover both **protocol-level** and **integration-level** security concerns.  
Use the relevant part depending on your project’s relationship with Aave:

| Type of Project | Use This Section | Description |
|------------------|------------------|-------------|
| **Forking or Modifying Aave V2 Code** | **V2 Checklist** | For teams building their own lending protocol using Aave’s architecture (e.g., Hundred, Agave). Focuses on core contract logic, health factors, liquidation math, and protocol invariants. |
| **Integrating with Aave (V3)** | **V3 Checklist** | For projects using Aave as liquidity source (aggregators, vaults, structured products, bots). Focuses on safe use of Pool API, E-Mode, isolation, siloed borrowing, and correct parameter handling. |

---

# **Aave V2 Checklist (Protocol-Level Forks)**

## 1. Core Architecture & Contract Logic
- [ ] Are `LendingPool`, `CollateralManager`, and all logic libraries immutable and version-locked?  
  Why: Prevents unintended upgrades or mismatched logic references during forks.  
- [ ] Are `ValidationLogic`, `GenericLogic`, and `ReserveLogic` consistently applied across deposit, borrow, repay, and liquidation operations?  
  Why: Ensures all core functions share consistent validation and accounting.  
- [ ] Are all `DebtTokens` and `aTokens` minted/burned 1:1 with underlying assets?  
  Why: Prevents accounting mismatches and overcollateralization bugs.  
- [ ] Are protocol parameters managed via a time-locked `Configurator` or multisig?  
  Why: Protects against governance-based or admin privilege abuse.  

## 2. Deposits & Withdrawals
- [ ] Does `validateDeposit` confirm reserves are active and not frozen before deposit?  
  Why: Prevents deposits into inactive or frozen assets.  
- [ ] Does `updateState()` recompute liquidity and borrow indices atomically per transaction?  
  Why: Avoids stale index data leading to rate calculation errors.  
- [ ] Does withdrawal logic (`validateWithdraw`) re-evaluate the user’s health factor via `balanceDecreaseAllowed()`?  
  Why: Prevents withdrawals that would reduce user safety below liquidation threshold.  
- [ ] Is dust properly handled on full withdrawals?  
  Why: Prevents collateral lockup or precision-based collateral flags.  
- [ ] Are withdrawals executed via CEI (Checks-Effects-Interactions) pattern before token transfers?  
  Why: Reduces reentrancy risk during withdrawals.  

## 3. Borrowing, Repayment & Interest Rates
- [ ] Are both stable and variable debt modes validated before minting debt tokens?  
  Why: Prevents invalid debt creation and ensures rate model consistency.  
- [ ] Does `validateBorrow()` enforce LTV and health factor checks using current oracle prices?  
  Why: Prevents borrowing based on stale or manipulated prices.  
- [ ] Does `repay()` burn the exact type and amount of debt tokens and mark user as inactive if cleared?  
  Why: Maintains debt state integrity and avoids residual liabilities.  
- [ ] Are slope parameters (`slope1`, `slope2`) and utilization breakpoints configured and tested per asset?  
  Why: Misconfigured curves cause unbalanced markets or frozen liquidity.  
- [ ] Is the `InterestRateStrategy` interface compatible across networks?  
  Why: Avoids cross-chain mismatches that can disable lending pools.  
- [ ] Are oracles properly integrated through `LendingRateOracle` with secure governance?  
  Why: Prevents reliance on unverified or outdated rates.  

## 4. Liquidations
- [ ] Verify liquidation threshold: up to **50%** of debt can be liquidated in Aave V2.  
  Why: Ensures liquidation math and incentives remain within safe limits.  
- [ ] Use `Pool.getUserAccountData(address user)` to fetch HF and collateral data.  
  Why: Provides accurate liquidation and risk metrics.  
- [ ] Set `debtToCover = uint(-1)` only when full liquidation is desired.  
  Why: Avoids over-liquidation and excessive collateral seizure.  
- [ ] Ensure state updates (`updateState`, `updateInterestRates`) precede token transfers.  
  Why: Prevents reserve manipulation during liquidation execution.  

## 5. Flash Loans & Empty Market Risk
- [ ] Does `validateFlashLoan()` verify repayment amount + premium?  
  Why: Prevents free flash loan exploits through fee bypass.  
- [ ] Are callbacks (`executeOperation`) properly sandboxed to prevent reentrancy?  
  Why: Avoids reentrancy loops triggered during flash loan execution.  
- [ ] Has precision loss or rounding in `cumulateToLiquidityIndex` been tested under empty markets?  
  Why: Empty market rounding errors can cause balance inflation.  
- [ ] Ensure protocol mints aToken in every new market upon initialization.  
  Why: Prevents manipulation in newly created empty pools.  

## 6. Token Compatibility & Reentrancy Prevention
- [ ] Are ERC777 / ERC677 tokens blocked or sandboxed during transfers?  
  Why: Tokens with hooks can invoke reentrancy during transfers.  
- [ ] Is CEI pattern enforced across all liquidity and swap operations?  
  Why: Prevents reentrancy through unsafe state updates.  
- [ ] Are SafeERC20 wrappers used consistently for transfers and approvals?  
  Why: Ensures safe handling of ERC20s that return false instead of reverting.  

## 7. Oracle & Price Feed Safety
- [ ] Are multiple oracle sources or fail-safes (Chainlink, fallback feeds) implemented?  
  Why: Reduces single-point failure in price feeds.  
- [ ] Is the system designed to pause or revert on abnormal oracle responses or delays?  
  Why: Protects against stale or incorrect prices during volatility.  
- [ ] Are health factor and collateral checks recalculated at execution block, not pending tx?  
  Why: Prevents race conditions and manipulation between mempool and block execution.  

## 8. External Integrations & Allowance Hygiene
- [ ] Are external calls (e.g., ParaSwap, swaps, repay adapters) validated and restricted?  
  Why: Prevents external call hijacking or logic bypass.  
- [ ] Are token allowances reset to zero after use?  
  Why: Avoids leftover approvals that attackers can drain.  
- [ ] Are return values from external contracts checked for success?  
  Why: Ensures integration safety and prevents silent failures.  

---

# **Aave V3 Checklist (Integration-Level)**

## 1. E-Mode (Efficiency Mode)
- [ ] Confirm asset correlation before enabling E-mode.  
  Why: Incorrect correlation setup can trigger liquidation cascades.  
- [ ] Ensure `Pool.setUserEMode(id)` reverts safely if unsupported tokens are borrowed or HF < 1.  
  Why: Prevents unsafe E-mode transitions under unhealthy collateral.  
- [ ] Use category `id = 0` to disable E-mode.  
  Why: Ensures user exits efficiency mode safely.  

## 2. Siloed Borrowing
- [ ] Use `AaveProtocolDataProvider.getSiloedBorrowing(asset)` to detect siloed assets.  
  Why: Prevents combining siloed and general assets in the same position.  

## 3. Isolation Mode
- [ ] Decode bits 212–251 from `Pool.getConfiguration(asset)` to read debt ceiling.  
  Why: Misreading configuration can allow excessive borrowing.  
- [ ] Restrict borrowing in isolation mode to governance-approved stablecoins only.  
  Why: Prevents high-volatility assets from bypassing isolation limits.  
- [ ] Ensure protocol logic does not mix isolated and non-isolated positions.  
  Why: Mixing positions invalidates debt ceiling enforcement.  

## 4. Credit Delegation
- [ ] Confirm delegation approval via `.approveDelegation(delegatee, amount)` on the debt token.  
  Why: Prevents unauthorized borrowing under another user’s credit.  
- [ ] Validate that `delegatee` uses `Pool.borrow()` under approved limits.  
  Why: Enforces delegation boundaries securely.  
- [ ] Document or enforce on-chain loan terms between delegator and delegatee.  
  Why: Reduces legal ambiguity in delegated lending.  

## 5. Supply & Withdraw Operations
- [ ] Use `Pool.supply(asset, amount, onBehalfOf, referralCode)` — ensure `referralCode = 0`.  
  Why: Referral code logic is deprecated and should remain inactive.  
- [ ] Confirm contract has sufficient approval on `amount` before supply.  
  Why: Prevents failed or partial deposits.  
- [ ] Implement permit signatures via `supplyWithPermit()` correctly with ERC-2612 compliance.  
  Why: Ensures gas-efficient deposits and signature safety.  
- [ ] Use `Pool.withdraw(asset, amount, to)` and validate that HF remains ≥ 1 after withdrawal.  
  Why: Prevents unsafe withdrawals leading to liquidation.  

## 6. Borrowing & Repayment
- [ ] Retrieve borrow rates via `AaveProtocolDataProvider.getReserveData(asset)` — check indexes 7 & 8.  
  Why: Provides accurate rate data for modeling.  
- [ ] Use `Pool.borrow()` with `onBehalfOf = msg.sender` if not using credit delegation.  
  Why: Prevents borrowing on unintended accounts.  
- [ ] Repay using `Pool.repay(asset, amount, rateMode, onBehalfOf)` — avoid null address inputs.  
  Why: Prevents accidental repayment to zero address.  
- [ ] Verify debt-token addresses via `getReserveTokensAddresses(asset)` for accurate delegation setup.  
  Why: Ensures delegation references correct token contract.  

## 7. Liquidations
- [ ] HF ≤ 0.95 → up to 100% liquidation; HF > 0.95 → up to 50%.  
  Why: Confirms liquidation logic respects Aave V3 parameters.  
- [ ] Verify liquidation interface: Ethereum: `Pool.liquidationCall()`, L2: `L2Pool.liquidationCall()`.  
  Why: Prevents integration errors between L1/L2 implementations.  
- [ ] Use Dune or subgraph data to simulate liquidation edge cases pre-deployment.  
  Why: Tests liquidation safety under real-world volatility.  

## 8. Configuration & Multi-Chain Deployment
- [ ] Verify all InterestRateStrategy and Configurator interfaces across deployments.  
  Why: Ensures uniform protocol logic across networks.  
- [ ] Ensure consistent risk parameters (reserveFactor, LTV, liquidation threshold) across chains.  
  Why: Avoids discrepancies that break protocol invariants.  

---

