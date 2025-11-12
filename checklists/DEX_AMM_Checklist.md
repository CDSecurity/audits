# DEX / AMM Security Checklist 

A focused checklist for teams building or auditing **Decentralized Exchanges (DEXs)** / **Automated Market Maker (AMM) protocols**.  
AMMs form the core mechanism behind most modern DEXs, handling price discovery, liquidity, and trade execution entirely on-chain.  
This checklist consolidates key security validations, design assumptions, and integration risks relevant to liquidity pools, swap logic, and oracle-linked markets.  


---

## 1. General Architecture & Design

- [ ] Is the DEX/AMM architecture designed with clear assumptions about liquidity flow, pool ownership, and governance?  
  Why: Unclear system boundaries allow logic inconsistencies and unintended privilege escalation.

- [ ] Are the protocol design and threat model documented and up to date?  
  Why: Builders and auditors must know intended invariants and actor powers.

- [ ] Are assumptions about external systems (EigenLayer, oracles, etc.) explicitly defined and verified?  
  Why: Prevents blind trust in dynamic external inputs.

- [ ] Are upgrade and governance paths time-locked and documented?  
  Why: Prevents accidental or malicious parameter changes.

- [ ] Are emergency controls (pause, recovery, upgrade) restricted to verified roles?  
  Why: Limits damage during exploits.

- [ ] Are different AMM types (constant product, stable swap, concentrated liquidity) implemented with clear separation?  
  Why: Mixing mathematical invariants leads to mispriced trades and liquidity errors.

- [ ] Does the AMM prohibit or carefully validate flash swaps or flash loans?  
  Why: Flash loans amplify almost all known AMM exploits.

---

## 2. Swap Logic & Parameters

- [ ] Is there deadline protection for swaps?  
  Why: Prevents MEV bots from executing delayed or front-run trades.

- [ ] Is hardcoded slippage avoided and user-defined limits enforced?  
  Why: Hardcoded slippage exposes users to high volatility losses.

- [ ] Is `minAmountOut` calculated before the swap and validated post-transaction?  
  Why: Prevents users from receiving fewer tokens than expected.

- [ ] Is slippage computed off-chain and passed on-chain safely?  
  Why: On-chain computation can be manipulated within the same block.

- [ ] Is the slippage parameter enforced at the final transfer step?  
  Why: Protects users from execution losses due to MEV or volatility.

- [ ] Are swaps executed atomically and checked for output consistency?  
  Why: Prevents partial executions that desync user balances or pool reserves.

---

## 3. Token Compatibility & Behavior

- [ ] Are varying token decimals handled correctly?  
  Why: Decimal mismatches corrupt swap math and LP accounting.

- [ ] Are fee-on-transfer tokens supported safely?  
  Why: Deducted transfer fees break constant-product invariants.

- [ ] Are rebasing tokens correctly synchronized with reserves?  
  Why: Rebasing changes balances unpredictably, desyncing reserves.

- [ ] Are hookable tokens (ERC777/677) restricted or handled safely?  
  Why: Hooks can trigger reentrancy and state inconsistencies.


---

## 4. Reserve & Pool Accounting

- [ ] Are pool reserves validated and reconciled after every transaction?  
  Why: Prevents balance desync that leads to solvency risk.

- [ ] Are rounding and invariant formulas tested across precision limits?  
  Why: Rounding drift causes pool imbalance and inaccurate pricing.

- [ ] Are product-constant equations (e.g., x * y = k) tested for precision and overflow?  
  Why: Critical for maintaining invariant consistency.

- [ ] Has any forked Uniswap or Curve code been diffed and reviewed?  
  Why: Prevents inherited vulnerabilities and logic drift.

---

## 5. Math & Precision Errors

- [ ] Are invariant and amplification formulas tested under small and large liquidity?  
  Why: Overflow/underflow often occurs in extreme ratios.

- [ ] Is fixed-point math implemented consistently with safe libraries (FullMath, SafeMath)?  
  Why: Inconsistent libraries cause rounding vulnerabilities.

- [ ] Are iterative math operations (root, exponent) bounded for gas and precision?  
  Why: Unbounded iterations can lead to DoS or inaccurate results.

- [ ] Are all balances using unsigned integers?  
  Why: Signed integers risk overflow on negative subtraction.

---

## 6. Reentrancy & CEI Compliance

- [ ] Is the Checks–Effects–Interactions (CEI) pattern enforced throughout?  
  Why: Prevents reentrancy between external calls and state updates.

- [ ] Are reserve updates executed before external token transfers?  
  Why: Prevents callback manipulation.

- [ ] Are ERC777/677 hooks sandboxed?  
  Why: Prevents read-only and write reentrancy attacks.

---

## 7. Price Manipulation & Oracles

- [ ] Does the AMM rely on internal spot prices without safeguards?  
  Why: Flash loan-based manipulation can drain pools or corrupt integrations.

- [ ] Are TWAP oracles implemented with block-based delay and cumulative tracking?  
  Why: TWAP mitigates flash attacks by averaging prices over time.

- [ ] Are oracle bounds validated (minAnswer < price < maxAnswer)?  
  Why: Protects against capped or stale data from Chainlink or Pyth.

- [ ] Are sandwich attacks mitigated via enforced minimum outputs or anti-MEV measures?  
  Why: Unbounded slippage enables front-run and back-run profit extraction.

---

## 8. Oracle Integration: General

- [ ] Are all oracle integrations reviewed and monitored?  
  Why: Oracles are a common single point of failure.

- [ ] Are oracle assumptions (pairs, latency, manipulation cost) defined?  
  Why: Hidden assumptions cause catastrophic edge cases.

- [ ] Are raw spot prices avoided as authoritative sources?  
  Why: Single-pool prices can be manipulated in one block.

- [ ] Is aggregation via TWAPs or multi-venue checks used?  
  Why: Aggregation increases manipulation cost.

- [ ] Are custom or homegrown oracles avoided unless essential?  
  Why: Custom oracle designs miss edge cases and attack vectors.

- [ ] Are fallback mechanisms active when feeds go invalid or unavailable?  
  Why: Prevents operations on broken data.

---

## 9. Chainlink Integration Checklist

- [ ] Are `.latestRoundData()` calls used instead of `.latestAnswer()`?  
  Why: Ensures timestamp and round metadata are included.

- [ ] Are price feed calls wrapped in try/catch?  
  Why: Prevents DoS on oracle revert.

- [ ] Does the protocol revert if latestPrice <= 0 or lastUpdatedAt == 0?  
  Why: Prevents invalid or uninitialized oracle data.

- [ ] Is a feed-specific `staleFeedThreshold` implemented?  
  Why: Different feeds update at different cadences.

- [ ] Is block.timestamp - lastUpdatedAt <= staleFeedThreshold enforced?  
  Why: Ensures freshness.

- [ ] Are decimals fetched dynamically?  
  Why: Prevents precision mismatch across feeds.

- [ ] Are all stablecoin price assumptions avoided?  
  Why: Stablecoins can depeg; assuming $1 is unsafe.

- [ ] If deployed on L2, is the sequencer uptime feed verified?  
  Why: Sequencer downtime freezes price updates.

---

## 10. Pyth Network Integration Checklist

- [ ] Are `updatePriceFeeds()` calls verified for signature validity?  
  Why: Prevents unauthorized updates.

- [ ] Are signatures verified to belong to authorized publishers?  
  Why: Ensures authenticity of price data.

- [ ] Does the protocol revert when price <= 0?  
  Why: Avoids corrupted or zero-value data.

- [ ] Is the exponent (decimal precision) validated?  
  Why: Incorrect exponents cause major price miscalculations.

- [ ] Is the confidence interval checked for reliability?  
  Why: Large intervals mean uncertain prices.

- [ ] Are staleness checks performed via `getPriceNoOlderThan()`?  
  Why: Prevents usage of stale data.

- [ ] Are `getPriceUnsafe()` and `getPriceNoOlderThan()` used atomically?  
  Why: Guarantees consistent price reads.

- [ ] Are fallback or alternative sources active if data is missing?  
  Why: Push-based systems can stall without updates.

---

## 11. Integration & Forking

- [ ] Has forked code (Uniswap, Curve, Balancer) been diffed using contract-diff.xyz?  
  Why: Forks often introduce subtle but critical logic divergences.

- [ ] Are integration libraries (SafeERC20, math libs, oracles) up-to-date and audited?  
  Why: Outdated dependencies reintroduce known vulnerabilities.

---

## 12. Access Control & Upgrades

- [ ] Are admin functions protected with timelocks and multisigs?  
  Why: Prevents single-signer compromise.

- [ ] Are initialization or setup functions callable only once?  
  Why: Prevents ownership hijacking.

- [ ] Are emergency and pause mechanisms audited to ensure they cannot drain funds?  
  Why: Prevents misuse of privileged functions.

---

## 13. Testing, Deployment & Parameters

- [ ] Are compiler versions pinned to stable audited releases?  
  Why: Avoids compiler-level vulnerabilities.

- [ ] Are invariant and fuzz tests implemented for swap logic and liquidity conservation?  
  Why: Ensures mathematical relationships (x*y=k) always hold.

- [ ] Are all parameters (decimals, feeds, pairs) double-checked before deployment?  
  Why: Misconfigured decimals can drain pools.

- [ ] Are off-chain keepers and bots treated as part of the threat model?  
  Why: Infrastructure failures can break on-chain logic.

---

Note: Every category of DeFi protocol — AMMs, DEXs, lending markets, LSDs, and others - ultimately relies on ERC20 token behavior. Many real-world exploits stem from non-standard token implementations. See [Weird ERC20 Implementations](https://github.com/d-xo/weird-erc20) for reference before integrating any token.



### Usage

This checklist is designed for internal and public security reviews of DEX and AMM systems.  
Use it to guide audits, continuous security monitoring, and public disclosure readiness.  
