# Lending and Money Markets Checklist

A unified audit checklist for **Lending and Money Market protocols** - including collateralized loans, credit markets, liquidation engines, and interest rate systems.  
Use it internally for protocol audits, due diligence, or comparative risk scoring.

---

## 1. General Design and Governance

- [ ] Is the protocol design and threat model documented and up to date?  
  Why: Auditors must understand intended invariants, actor powers, and assumptions.  

- [ ] Are assumptions about external systems (e.g., oracles, L2 sequencers, aggregators) explicitly defined and verified?  
  Why: Prevents reliance on unverified external data or infrastructure.  

- [ ] Are upgrade and governance paths time-locked and fully auditable?  
  Why: Avoids malicious or rushed parameter or code changes.  

- [ ] Are emergency controls (pause, recovery, upgrade) restricted to verified roles?  
  Why: Prevents unauthorized control over funds during critical incidents.  

- [ ] Are isolated markets, if used, properly segregated by collateral and debt token lists?  
  Why: Prevents cross-market contamination and insolvency propagation.  

---

## 2. Initialization and Market Setup

- [ ] Are new markets initialized with non-zero share supply?  
  Why: Empty markets can cause rounding exploits and exchange-rate inflation.  

- [ ] Is the deployment script verified to mint and burn bootstrap liquidity correctly?  
  Why: Prevents zero-supply arithmetic edge cases.  

- [ ] Are fixed-point decimals standardized across all supported tokens?  
  Why: Prevents scaling errors and invisible arbitrage via decimal mismatches.  

- [ ] Are rounding-sensitive variables (exchangeRate, index, pseudoTotalPool) monitored on-chain?  
  Why: Detects manipulation or silent accounting drift.  

- [ ] Is the audit scope aligned with the latest protocol specification?  
  Why: Ensures that new logic paths and invariants are correctly verified.  

---

## 3. Accounting, Math, and Invariants

- [ ] Are all exchange rate and index calculations independent of raw token balances?  
  Why: Prevents manipulation through direct token transfers (“force feeding”).  

- [ ] Are rounding operations explicitly audited for direction (divUp vs divDown)?  
  Why: Wrong rounding leads to supply inflation or silent share burning.  

- [ ] Are debt, collateral, and pool invariants continuously fuzz-tested?  
  Why: Finds rare precision or rounding-related logic failures.  

- [ ] Are arithmetic assumptions re-audited after new pool types or flash features are added?  
  Why: New logic paths may break existing invariants.  

- [ ] Are partial withdrawals or single-user pools bounded or restricted?  
  Why: Single-user ownership can break mint/burn invariants and donation logic.  

- [ ] Are tick or price boundaries rounded consistently?  
  Why: Misrounded tick math can lead to liquidity double-counting.  

---

## 4. Borrowing and Repayment

- [ ] Is interest correctly included in Loan-to-Value (LTV) and health calculations?  
  Why: Ignoring accrued interest underestimates borrower risk.  

- [ ] Can liquidation and repayment be enabled or disabled inconsistently?  
  Why: Mismatched toggle states can trap funds or enable unfair liquidations.  

- [ ] Can borrowers repay loans across multiple active positions accurately?  
  Why: Prevents overpayment or miscrediting of repayments.  

- [ ] Can repayment and collateral release occur atomically?  
  Why: Ensures consistent borrower and lender accounting.  

- [ ] Are repayments validated for non-zero lender addresses?  
  Why: Prevents repayment funds from being sent to `address(0)`.  

- [ ] Can loans enter a state where repayment becomes impossible?  
  Why: Prevents borrower or lender lockout due to logic bugs or governance misconfigurations.  

- [ ] Is interest accrual paused when the protocol itself is paused?  
  Why: Continuous accrual during pause can make users instantly liquidatable when resumed.  

---

## 5. Liquidation Logic

- [ ] Does liquidation only occur after valid default or undercollateralization?  
  Why: Borrowers must not be liquidated pre-default or before repayment deadlines.  

- [ ] Does the protocol handle partial and full liquidation correctly?  
  Why: Incorrect sequencing can worsen borrower health or over-liquidate.  

- [ ] Is partial liquidation supported for large positions?  
  Why: Prevents whales from becoming unliquidatable due to gas or liquidity limits.  

- [ ] Does partial liquidation always improve borrower health score?  
  Why: Ensures progressive risk reduction instead of recursive liquidations.  

- [ ] Are liquidator rewards correctly calculated and paid first when collateral is insufficient?  
  Why: Keeps liquidation incentives aligned even in low-collateral conditions.  

- [ ] Is there a minimum liquidation incentive ensuring gas profitability?  
  Why: Prevents buildup of bad debt due to unprofitable positions.  

- [ ] Can borrowers front-run liquidations to reset or manipulate cooldowns?  
  Why: Front-running liquidation conditions can cause DoS against honest liquidators.  

- [ ] Can users self-liquidate for profit using oracle manipulation or flash loans?  
  Why: Allows exploiters to drain value via timing-based arbitrage.  

- [ ] Are liquidation rewards and fees consistent with token decimals?  
  Why: Mismatched precision distorts payouts and accounting.  

- [ ] Is there a defined LTV gap between borrowing and liquidation thresholds?  
  Why: Eliminates edge conditions that trigger instant liquidation on borrow.  

- [ ] Does liquidation iterate safely without unbounded loops?  
  Why: Unbounded iteration over user positions can cause out-of-gas reverts.  

- [ ] Can liquidation proceed when only one borrower exists?  
  Why: Single-user logic edge cases often revert due to array indexing errors.  

- [ ] Does debt stop accruing interest during liquidation?  
  Why: Borrowers shouldn’t accumulate interest once liquidation starts.  

- [ ] Can the liquidator specify minimum output or slippage limits?  
  Why: Protects liquidators from sandwich or MEV attacks during swaps.  

- [ ] Are all user positions incentivized fairly for liquidation?  
  Why: Ensures small and large liquidations occur promptly.  

---

## 6. Collateral Management

- [ ] Are collateral assets verified against actual borrower positions before liquidation?  
  Why: Prevents liquidation of unrelated or unbacked assets.  

- [ ] Are collateral tokens validated and initialized correctly (no zero-address inputs)?  
  Why: Avoids invalid token references that can bypass checks.  

- [ ] Can borrowers overwrite or reset validated collateral to zero?  
  Why: Removes effective loan backing while keeping loan valid.  

- [ ] Is collateral in yield vaults or LPs properly seized during liquidation?  
  Why: Collateral embedded in other contracts must remain claimable.  

- [ ] Are non-18 decimal tokens handled correctly in collateral math?  
  Why: Prevents scaling and reward miscalculation.  

- [ ] Is earned yield or user PNL included in collateral valuation?  
  Why: Excluding accrued profit leads to unfair or premature liquidation.  

- [ ] Is open PNL used consistently in collateral evaluation?  
  Why: Prevents under- or overestimation of user risk.  

---

## 7. Oracle and Pricing Integrity

- [ ] Are all oracle sources immutable and validated externally?  
  Why: Prevents governance or admin abuse of price feeds.  

- [ ] Does the protocol use Chainlink or equally decentralized sources?  
  Why: Reduces manipulation via flashloaned on-chain reserves.  

- [ ] Are fallback oracles (TWAP, custom) used only for liquid pairs?  
  Why: Prevents low-liquidity manipulation via reserve spoofing.  

- [ ] Has the cost of oracle manipulation been analyzed relative to potential gain?  
  Why: TWAP attacks remain viable if cost < liquidation profit.  

- [ ] Are oracles for LP/vault tokens derived securely (TVL ÷ totalSupply)?  
  Why: Prevents overvaluation of collateral via inflated TVL.  

- [ ] Are composite or derivative asset oracles cross-verified?  
  Why: Derived asset pricing often hides manipulation paths.  

- [ ] Are oracle updates atomic with liquidation checks?  
  Why: Prevents stale data from triggering incorrect liquidations.  

- [ ] Are L2 sequencer downtimes handled with a grace period before liquidation resumes?  
  Why: Allows users to restore collateral after oracle outages.  

---

## 8. Token Safety and Reentrancy

- [ ] Are reentrancy guards present on all external state-changing functions?  
  Why: Prevents borrow or liquidation logic from being reentered mid-call.  

- [ ] Are ERC777/677 or other hookable tokens restricted from use as collateral or debt?  
  Why: Hook callbacks can bypass LTV and repayment logic.  

- [ ] Are CEI (Checks-Effects-Interactions) patterns consistently applied?  
  Why: Prevents reentrancy from external token interactions.  

- [ ] Are ETH receive/fallback functions isolated from protocol logic?  
  Why: ETH hooks behave like reentrancy vectors if unchecked.  

- [ ] Has the protocol been tested assuming all tokens are malicious or hookable?  
  Why: Reveals reentrancy or callback attack surfaces.  

---

## 9. Economic Safety and Incentive Design

- [ ] Are utilization-based interest rate formulas capped and bounded?  
  Why: Prevents overflows and runaway rates under high utilization.  

- [ ] Are max leverage and recursive borrow depth limited?  
  Why: Uncapped recursive leverage can cascade insolvency.  

- [ ] Are liquidation fees proportional to liquidator profit, not seized collateral?  
  Why: Keeps incentives healthy and prevents unprofitable liquidations.  

- [ ] Does the protocol implement bad debt handling or insurance funds?  
  Why: Prevents permanent insolvent positions from freezing system liquidity.  

- [ ] Are yield and funding fees refreshed before liquidation checks?  
  Why: Ensures consistent solvency calculation.  

- [ ] Does partial liquidation correctly cover bad debt?  
  Why: Missing coverage allows bad debt to accumulate unseen.  

---

## 10. Testing and Continuous Monitoring

- [ ] Are invariants and health factor calculations covered by fuzz and differential tests?  
  Why: Detects cross-pool or multi-asset edge failures.  

- [ ] Are oracles and LTV logic continuously monitored on-chain?  
  Why: Catches stale price feeds and misconfigurations early.  

- [ ] Are liquidation and interest parameters logged for forensic analysis?  
  Why: Enables root-cause diagnosis of liquidation anomalies.  

- [ ] Are governance and pause events emitted on every configuration change?  
  Why: Ensures transparency and traceability of protocol state changes.  

---

Note: Every category of DeFi protocol — AMMs, DEXs, lending markets, LSDs, and others - ultimately relies on ERC20 token behavior. Many real-world exploits stem from non-standard token implementations. See [Weird ERC20 Implementations](https://github.com/d-xo/weird-erc20) for reference before integrating any token.

### Usage

This checklist is designed for internal and public security reviews oflending systems.  
Use it to guide audits, continuous security monitoring, and public disclosure readiness. 
